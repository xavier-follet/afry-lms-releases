# AFRY LMS — Production Deployment

## Architecture

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 260}}}%%
flowchart TB
    user(["Browser"])

    subgraph tls["TLS termination: pick one"]
        direction LR
        proxy["<b>Caddy or Nginx</b><br/>on the host"]
        agw["<b>Azure Application Gateway</b><br/>in front of the VM"]
    end

    subgraph host["Linux host: Docker Compose"]
        frontend["<b>afry_lms_frontend</b><br/>Nginx :8080<br/><i>SPAs, uploaded files, /api proxy;<br/>package files on the content host</i>"]
        backend["<b>afry_lms_backend</b><br/>Node 24 / Express :3200"]
        db[("<b>afry_lms_db</b><br/>PostgreSQL 18 :5432")]
        uploads[("<b>uploads_data</b><br/>volume")]
        migrate["<b>migrate</b><br/><i>one-shot, every up</i>"]
    end

    user -- "HTTPS :443, both names" --> proxy
    user -- "HTTPS :443, both names" --> agw
    proxy -- "HTTP 127.0.0.1:8081" --> frontend
    agw -- "HTTP private IP:8081" --> frontend
    frontend -- "/api/" --> backend
    frontend -. "read-only" .-> uploads
    backend --> uploads
    backend --> db
    migrate -. "schema" .-> db

    classDef edge fill:#fdebd3,stroke:#c77d1a,color:#3d2400
    classDef app fill:#dfe9ff,stroke:#3a6bd6,color:#0b2a66
    classDef data fill:#dcf2e3,stroke:#2e9a59,color:#0d3b20
    classDef job fill:#f1f1f1,stroke:#8a8a8a,color:#333,stroke-dasharray:4 3
    classDef client fill:#ffffff,stroke:#555,color:#111
    class proxy,agw edge
    class frontend,backend app
    class db,uploads data
    class migrate job
    class user client
    style tls fill:none,stroke:#c77d1a,stroke-dasharray:5 4
    style host fill:none,stroke:#8a8a8a
```

The three long-running containers and the one-shot `migrate` container are pulled from GitHub Container Registry (GHCR). **No source code or repository access is needed on the server.**

---

## Prerequisites

- Linux server (Ubuntu 26.04 LTS recommended)
- Two DNS names with an A record pointing to the server IP: the app's (`yourdomain.com`) and the content host packages are served from (`content.yourdomain.com`). See [Content origin](#content-origin)
- Port 443 open in the firewall (`APP_PORT`, the port the AFRY LMS container binds to, is localhost-only and must not be exposed externally)
- Docker 24+ with the Compose plugin
- `sudo` rights on the server
- The image pull token from the AFRY LMS team (see [step 4](#4-authenticate-with-ghcr)). No GitHub account is needed
- Outbound HTTPS to `github.com` and `*.githubusercontent.com` (release files) and `ghcr.io` (images), or a registry of your own: see [Using your own registry](#using-your-own-registry). The full list, including email, AI and certificates, is in [Server requirements](./server-requirements.md#outbound)
- `curl` and `openssl` (both preinstalled on Ubuntu), plus `gpg` and `rsync` for the [backup scripts](./backup-and-recovery.md)
- A separate data disk, if you follow [Encryption at rest](#encryption-at-rest). Plan it before the first start: moving Docker's data afterwards means migrating the volumes

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # re-login after this
```

---

## First-time setup

### 1. Create a working directory

```bash
mkdir -p /srv/afry_lms && cd /srv/afry_lms
```

### 2. Download the release files

The release files are public. No GitHub account and no login are needed. They are published at [github.com/xavier-follet/afry-lms-releases](https://github.com/xavier-follet/afry-lms-releases/releases/latest):

- `docker-compose.yml`
- `env.prod.example`
- `backup.sh`, `restore.sh`, `backup-healthcheck.sh`, `verify-backup.sh` (place these in `/srv/afry_lms/scripts/` and `chmod +x` — see the [Backup & Recovery runbook](./backup-and-recovery.md))
- `rotate-secret.sh`, in the same directory (see [Rotating secrets](#rotating-secrets))
- `digests.txt`, `cosign.pub` and one `*.cdx.json` SBOM per image, for checking the images (see [Verifying the images](#verifying-the-images))

```bash
BASE=https://github.com/xavier-follet/afry-lms-releases/releases/latest/download
mkdir -p scripts
for f in docker-compose.yml env.prod.example; do curl -fsSLo "$f" "$BASE/$f"; done
for f in backup.sh restore.sh backup-healthcheck.sh verify-backup.sh rotate-secret.sh; do curl -fsSLo "scripts/$f" "$BASE/$f"; done
chmod +x scripts/*.sh
```

To fetch a specific release instead of the newest, replace `latest/download` with `download/v1.2.3`.

### 3. Configure environment variables

```bash
cp env.prod.example .env
nano .env
```

See [Environment variables](#environment-variables) for the full reference. The minimum required values:

```env
GITHUB_REPOSITORY=xavier-follet/afry-lms
POSTGRES_PASSWORD=<strong random password>
JWT_SECRET=<output of: openssl rand -hex 32>
FILE_TOKEN_SECRET=<output of: openssl rand -hex 32>
SCORM_CONTEXT_SECRET=<output of: openssl rand -hex 32>
CORS_ORIGIN=https://yourdomain.com
CONTENT_HOST=content.yourdomain.com
# Needed for email or AI. Leave it empty if you use neither, never as the placeholder
SMTP_ENCRYPTION_KEY=<output of: openssl rand -hex 32>
```

> **Upgrading an existing install?** Add `SCORM_CONTEXT_SECRET` to `.env` before running `docker compose up -d` on the new release. Compose stops with an error if it is missing, and nothing on the running stack changes.
>
> Set `CONTENT_HOST` too, with a DNS record and a reverse-proxy site for it: see [Content origin](#content-origin). Without it, SCORM and HTML packages run on the app's own address, with the viewer's session in their reach.
>
> The backend also refuses to start on a placeholder, short or malformed secret, or on the default database password. If it keeps restarting after an upgrade, `docker compose logs backend` names the variable. See [Startup checks](#startup-checks). The fix depends on which one it is:
>
> - **`POSTGRES_PASSWORD`** is `secret` or the placeholder: run `scripts/rotate-secret.sh POSTGRES_PASSWORD`. Editing `.env` alone breaks the database connection.
> - **`SMTP_ENCRYPTION_KEY`** is the `CHANGE_ME_64_HEX_CHARS` placeholder: replace it in `.env` with `openssl rand -hex 32`, or empty it if you use neither email nor AI. Nothing can have been stored under the placeholder, so nothing is lost. Do not use the script for this one.
> - **`JWT_SECRET`, `FILE_TOKEN_SECRET` or `SCORM_CONTEXT_SECRET`** is short or the placeholder: run `scripts/rotate-secret.sh <NAME>`. Nobody is signed out.

### 4. Authenticate with GHCR

The images are private. The AFRY LMS team sends you an access token for pulling them. You do not need a GitHub account.

```bash
echo YOUR_TOKEN | docker login ghcr.io -u afry-lms-customer --password-stdin
```

The username is not checked; any non-empty value works. This is a one-time step. Docker stores the credential in `~/.docker/config.json`.

If the server pulls from your own registry instead, log in to that one: see [Using your own registry](#using-your-own-registry).

### 5. Pull images and start

To check that the images are the ones AFRY LMS signed before you run them, follow
[Verifying the images](#verifying-the-images) first.

```bash
docker compose up -d
```

Then pull the small `alpine` image that `backup.sh` and `restore.sh` copy the uploads volume
with. The scripts pin it by digest, and pulling it now means a backup never has to reach
Docker Hub (see [The helper image](./backup-and-recovery.md#the-helper-image), which also
covers hosts without internet):

```bash
docker pull "$(grep -o 'alpine:[0-9.]*@sha256:[0-9a-f]*' scripts/backup.sh)"
```

On first start, PostgreSQL creates an empty database. The `migrate` service then builds the schema by applying every migration in order (starting from `0001_initial_schema.sql`), and the backend starts only after it has finished. There is no seed data: the backend creates the superadmin from `SUPERADMIN_EMAIL` / `SUPERADMIN_PASSWORD` if none exists yet. That password works once: the first sign-in with it goes straight to **Set a new password**, and the account must choose its own before it gets in. The backend log says `Superadmin created` only when it actually created the account; if the email already belongs to a user who is not a super-user, it logs a warning and creates nothing. Every later `docker compose up` runs `migrate` again, which applies only the migrations that are new.

### 6. Verify

```bash
docker compose ps -a                   # postgres, backend, frontend "Up"; migrate "Exited (0)"
curl http://localhost:8081/api/version # → {"app":"v1.2.3","db":"…","externalFonts":"bunny"}
curl -f http://localhost:8081/api/health # → {"status":"ok"}
```

`/api/version` always answers 200; `"db": null` there means the backend cannot reach the database. For a check that depends only on the status code (a load-balancer probe, a monitoring tool), use `/api/health`: 200 when the database answers, 500 when it does not or takes more than 3 seconds.

The backend's port 3200 is not published to the host, so check it through the frontend on `APP_PORT`. That also proves the Nginx-to-backend proxy works. A `migrate` container with a non-zero exit code means a migration failed, and the backend will not have started: see `docker compose logs migrate`.

---

## Content origin

Imported SCORM courses and HTML packages are uploaded by course producers, and they run
whatever script their author wrote. AFRY LMS serves them from a second host name of their
own, `CONTENT_HOST`, so that the browser keeps them apart from the app: a package can script
its own frames, which authoring tools such as Rise and Storyline need, but it cannot read
the session of the learner viewing it, call the API as them, or reach the page around it.

- **Pick a name under the app's own domain**, e.g. `content.yourdomain.com` for
  `yourdomain.com`. Browsers then treat the two as one site, and packages keep their own
  cookies. Two unrelated domains work as well, but some browsers then refuse packages'
  cookies.
- **Point it at the same server** with an A record, and give your reverse proxy a site and
  certificate for it, as in the options below. Nothing else changes: the same containers
  serve both names, and the frontend's Nginx answers on the content host with package files
  only, never the app.
- **Azure-provided names** (`<label>.<region>.cloudapp.azure.com`) cannot carry a second
  name beneath them; use a domain of your own for both names.

Set it in `.env` as the host name alone, without `https://`:

```env
CONTENT_HOST=content.yourdomain.com
```

> **Leaving `CONTENT_HOST` empty is a security decision**, and so is setting it to the app's
> own host name (with any port: ports do not separate them). Packages are then served from
> the app's own address, where any script inside one can read the session of whoever views
> it and act in the platform as them: an author with upload rights could take over a
> super-user's account by getting them to open a page. Choose this only for an installation
> where every package uploaded to it comes from someone you trust. The backend logs a
> warning at every start while it is empty.

---

## Reverse proxy + HTTPS

Options A and B run on the server itself. Option C uses Azure Application Gateway
instead, for a VM in Azure.

Caddy and Nginx both need port 80 for themselves — the HTTP listener, the ACME
HTTP-01 challenge, and the HTTP-to-HTTPS redirect — so `APP_PORT` must be
something other than 80 before you install either. The shipped
`env.prod.example` already defaults to `APP_PORT=8081`; the examples below
match that. If you changed `APP_PORT` in `.env`, update `localhost:8081` in
the examples to match.

Each option below adds exactly one proxy in front of the app, which is what the
default `TRUST_PROXY=2` assumes. If you put another layer in front (a CDN, a cloud
load balancer), raise it. See [Environment variables](#environment-variables).

### Option A — Caddy (recommended)

Caddy handles certificate provisioning automatically.

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy
```

`/etc/caddy/Caddyfile`, both names in one site:
```
yourdomain.com, content.yourdomain.com {
    reverse_proxy localhost:8081
}
```

```bash
sudo systemctl enable --now caddy
```

### Option B — Nginx + Certbot

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d content.yourdomain.com
```

One certificate covers both names; Certbot keeps it under the first one.

`/etc/nginx/sites-available/afry_lms`:
```nginx
server {
    listen 80;
    server_name yourdomain.com content.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com content.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass              http://localhost:8081;
        proxy_set_header Host   $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 10240M;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/afry_lms /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Option C — Azure Application Gateway

For a VM in Azure. Application Gateway takes Caddy's place: it holds the
certificate, terminates TLS and forwards plain HTTP to the VM over the private
network. Nothing is installed on the VM itself. The VM needs no public IP, and the
domain's DNS record points at the gateway's public IP rather than the VM.

**1. Let the gateway reach the app.** The shipped compose file binds the app to
`127.0.0.1`, which only the VM itself can reach. Add a second binding on the VM's
private IP in an override file, which Compose loads automatically. Do not edit
`docker-compose.yml`: it is replaced with each release. `backup.sh`, `restore.sh` and
`rotate-secret.sh` load the override file too, so a restore or a secret rotation keeps the
binding; each logs the compose files it uses when it starts.

`/srv/afry_lms/docker-compose.override.yml`:
```yaml
services:
  frontend:
    ports:
      - "10.0.1.4:8081:8080"   # the VM's private IP
```

```bash
docker compose up -d
curl http://10.0.1.4:8081/api/version   # from the VM
```

**2. Restrict it.** In the VM's network security group, allow inbound TCP 8081
from the Application Gateway subnet only.

**3. Configure the gateway** (v2 SKU, in its own subnet of the same VNet):

| Setting | Value | Why |
|---|---|---|
| Listener | HTTPS 443, multi-site, one per name (your domain and `CONTENT_HOST`), each with its certificate, or one certificate covering both | Replaces Caddy |
| Redirect | HTTP 80 to HTTPS 443 | |
| Backend pool | The VM's private IP | |
| Backend settings | HTTP, port 8081, **timeout 120 s** | Default 20 s is too short for SCORM imports and AI calls |
| Override with new host name | **No** (default) | The backend must see the name the browser used: it tells the app and the content host apart by it |
| Cookie-based affinity | Disabled | Tokens, no server session |
| Health probe | `/api/health`, host = your domain, 200–399 | 200 only when the backend reaches the database, 500 otherwise. Not `/api/version`: it always answers 200 |

**If the WAF is enabled**, raise its file upload limit to at least 200 MB (the
largest SCORM and package import), and check its request body size limit:
saving a page in the editor can send a large JSON body. Rich-text HTML and SCORM
runtime traffic are likely to trip OWASP rules, so run the WAF in Detection mode
during the pilot, add exclusions for what it flags, and only then switch to
Prevention.

Keep `TRUST_PROXY=2`. The backend strips the `:port` Application Gateway adds to
the client address on its own. If Azure Front Door sits in front of the gateway,
set `TRUST_PROXY=3`.

**End-to-end TLS.** The setup above sends plain HTTP inside the VNet. If your
policy requires TLS between the gateway and the VM as well, run Caddy or Nginx on
the VM with a certificate the gateway trusts, point the backend settings at it
over HTTPS, and raise `TRUST_PROXY` by one.

---

## Encryption at rest

AFRY LMS stores personally identifiable information (PII) — user names, email addresses, and training progress — in PostgreSQL. The database runs in a Docker container backed by a named volume on the host filesystem. **Unless that host filesystem is encrypted, PII on disk is readable by anyone with physical or root access to the machine.**

Configure LUKS on a dedicated data disk before starting Docker so the database volume lands on encrypted storage.

```bash
sudo apt install -y cryptsetup

# Confirm device name — replace /dev/sdb below with your data disk
lsblk

sudo cryptsetup luksFormat /dev/sdb
sudo cryptsetup luksOpen /dev/sdb afry_lms_data
sudo mkfs.ext4 /dev/mapper/afry_lms_data

sudo mkdir -p /mnt/afry_lms
sudo mount /dev/mapper/afry_lms_data /mnt/afry_lms
```

Point Docker's data directory at the encrypted mount (run this before the first `docker compose up`):

```bash
sudo systemctl stop docker

sudo mkdir -p /mnt/afry_lms/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "data-root": "/mnt/afry_lms/docker"
}
EOF

sudo systemctl start docker
```

For unattended reboots, add an entry to `/etc/crypttab` and store the passphrase in a keyfile. See the [Ubuntu full-disk encryption guide](https://ubuntu.com/server/docs/full-disk-encryption) for the complete procedure.

---

## Updating the application

Each GitHub Release publishes updated images. Versions read `vMAJOR.MINOR.REVISION`:

- **A new minor version** (`v0.3.0`) brings changes to the application: features, fixes,
  database migrations, `.env` variables.
- **A new revision** (`v0.3.1`) is the same application as the minor version before it, with
  either:
  - **refreshed base images and operating-system packages**, published monthly so security
    fixes from upstream reach your installation. Its release notes say "No changes you would
    notice in the application", and it has no migrations and no `.env` changes. A month in
    which no base image or package changed has no such release.
  - **a hotfix**: one urgent fix, listed in its release notes.

Only the newest minor version receives these revisions. If you run an older one, upgrade to
the newest to get them.

To deploy a new version (unless its release
notes say it moves to a new PostgreSQL major version: then follow
[PostgreSQL major-version upgrades](#postgresql-major-version-upgrades) instead):

1. **Read the release notes** of every release between the one you run and the one you
   are moving to, in the [releases list](https://github.com/xavier-follet/afry-lms-releases/releases)
   or its Atom feed (`/releases.atom` at the same address). Each one opens with:

   - **What changed**: new features, fixes, and changes in behaviour.
   - **Upgrade**: steps to take before upgrading, `.env` variables that are new or no
     longer used (`env.prod.example` explains each one), and the database migrations the
     release applies on startup. A migration that drops a table or column carries a
     warning: that data comes back only from the backup in the next step.

   The notes list what changed. They are not a guarantee that an upgrade is safe, which
   is why the backup comes first.

2. **Take a backup first**, while the current version is still running (the script
   checks that PostgreSQL is up before it starts):

   ```bash
   sudo /srv/afry_lms/scripts/backup.sh
   echo $?   # must print 0, and the last log line must read "backup <timestamp> complete"
   ```

   Do not continue on any other exit code. Migrations only move forward and some of
   them drop columns or tables, so this backup is the only way back if the upgrade has
   to be undone (see [Rolling back](#rolling-back)). For the script's settings and
   exit codes, see [Running a backup](./backup-and-recovery.md#4-running-a-backup).
   To also prove the backup restores, run `sudo /srv/afry_lms/scripts/verify-backup.sh latest`
   ([Verifying a backup](./backup-and-recovery.md#10-verifying-a-backup-restore-drill)).

3. **Pull and restart:**

   ```bash
   # Pull and restart with the latest images
   docker compose pull
   docker compose up -d

   # Pin to a specific release
   APP_VERSION=v1.2.3 docker compose pull
   APP_VERSION=v1.2.3 docker compose up -d
   ```

   If the `docker-compose.yml` itself changed in the release, download it again before running the above.
   To check the new images first, see [Verifying the images](#verifying-the-images). If you pin
   images by digest, update `docker-compose.override.yml` with the new digests, or the pull
   keeps the old release.

### Rolling back

There is no downgrade. Migrations only move forward, and the ones that drop data cannot
be reversed, so the way back is the backup taken before the upgrade plus the previous
`APP_VERSION`. The backup's manifest (`/srv/backups/afry_lms-<timestamp>.manifest`)
records the image it was taken from on its `app_image=` line. Anything entered after
that backup is lost.

Never start an older version against a database a newer release has already migrated:
the older code does not know the newer schema. The stack refuses to. Set `APP_VERSION`
back without restoring and the `migrate` one-shot exits 1 before changing anything, the
backend does not start, and `docker compose logs migrate` reads:

```text
Refusing to start: the database was migrated by a newer release. It holds
0054_example.sql, which this image does not contain (its last migration is 0053_system_message.sql). ...
```

The database is unharmed: put `APP_VERSION` back to the newer version and `docker compose up -d`
brings the site back as it was. To roll back instead:

1. Read the version the backup was taken on:

   ```bash
   grep app_image= /srv/backups/afry_lms-<timestamp>.manifest
   # app_image=ghcr.io/<org>/<repo>/backend:v1.2.3
   ```

   The tag after the last `:` is the version. If it reads `latest`, the manifest cannot tell
   you which release that was, which is one more reason to pin `APP_VERSION`.

2. Set `APP_VERSION` in `.env` to that version and pull its images:

   ```bash
   docker compose pull
   ```

3. Restore the backup. This replaces the database and the uploaded files, then starts the
   stack on the older images:

   ```bash
   sudo /srv/afry_lms/scripts/restore.sh <timestamp>
   ```

   The database is replaced whole, so tables a newer release added do not linger. See
   [Restore](./backup-and-recovery.md#9-restore).

### PostgreSQL major-version upgrades

A release that moves the `db` image to a new PostgreSQL major version says so in its
release notes, under a heading such as **PostgreSQL 18 to 19**. Do not install that release
with the plain `pull` and `up` above.

A PostgreSQL major version cannot read the data files of the previous one. Started on the
existing volume, the new `db` image finds the old version's data and refuses to start:
`docker compose ps` shows it restarting, `migrate` and the backend wait for it, and the site
answers 502 Bad Gateway. `docker compose logs postgres` says
`there appears to be PostgreSQL data in: /var/lib/postgresql/18/docker`. Nothing is damaged:
set `APP_VERSION` back to the previous version, run `docker compose up -d`, and follow the
steps below.

The procedure moves the data with a dump and a restore into a new volume, using the
backup scripts. The old volume is not touched, so until you remove it, going back is a
matter of switching back. The application is down from step 1 until the checks in
step 7 pass. Plan for about twice as long as your nightly backup takes (the timestamps in
`/srv/backups/backup.log` show it).

The commands run in `/srv/afry_lms`. They use the default database user and name,
`afry_lms` and `afry_lms_db`: use your `POSTGRES_USER` and `POSTGRES_DB` if `.env` sets
them. Check that the disk has room for a second copy of the database:
`docker system df -v | grep postgres_data` shows the current volume's size.

1. **Stop the application and back up.** Stopping `frontend` and `backend` first means
   nothing is written after the backup. The database stays up, because the backup needs it.

   ```bash
   cd /srv/afry_lms
   docker compose stop frontend backend
   docker compose ps   # must list afry_lms_db only
   sudo /srv/afry_lms/scripts/backup.sh
   echo $?   # must print 0
   ```

2. **Record the row counts** of every table, to compare after the restore:

   ```bash
   docker compose exec -T postgres psql -U afry_lms -d afry_lms_db -tA > ~/rows-before.txt <<'SQL'
   SELECT table_name || '=' || (xpath('/row/c/text()', query_to_xml(format('SELECT count(*) AS c FROM %I', table_name), false, true, '')))[1]
   FROM information_schema.tables WHERE table_schema = 'public' AND table_type = 'BASE TABLE' ORDER BY 1;
   SQL
   ```

3. **Check that the backup restores into the new version** before you change anything.
   `verify-backup.sh` restores it into a throwaway container of the image you name and
   leaves the live data alone (see
   [Verifying a backup](./backup-and-recovery.md#10-verifying-a-backup-restore-drill)).
   Setting `APP_VERSION` for this one command makes it use the new release's `db` image:

   ```bash
   sudo APP_VERSION=<new version> /srv/afry_lms/scripts/verify-backup.sh latest
   ```

   It must end with a line starting `OK:`. If it does not, stop here: run
   `docker compose up -d` and you are back on the current version.

4. **Switch to the new release.** Download the new release's `docker-compose.yml`, set
   `APP_VERSION` in `.env` to the new version, then:

   ```bash
   docker compose down     # never with -v: that deletes the volumes
   docker compose pull
   ```

5. **Give the database a new volume.** Add this to
   `/srv/afry_lms/docker-compose.override.yml`, creating the file if there is none, and
   name the volume after the new major version:

   ```yaml
   volumes:
     postgres_data:
       name: afry_lms_postgres_data_pg19
   ```

   `docker compose config | grep postgres_data_pg` should now print that name. If the file
   already has a `volumes:` block from an earlier upgrade, change the name in it rather than
   adding a second block.

6. **Start the new database and restore into it.** `restore.sh` then starts `migrate`, the
   backend and the frontend:

   ```bash
   docker compose up -d --wait postgres
   sudo /srv/afry_lms/scripts/restore.sh latest --db-only --yes
   ```

   The script's last check runs before the frontend is ready, so a warning that it did not
   respond yet is expected.

7. **Check.** All of these must hold before anyone uses the application:

   ```bash
   curl -f http://localhost:8081/api/health          # {"status":"ok"}
   curl http://localhost:8081/api/version            # "db" names the newest migration
   docker compose exec postgres psql -U afry_lms -d afry_lms_db -tAc 'SHOW server_version'
   ```

   A 502 in the first minute only means the backend is still starting: try again.

   Run the step 2 command again with `~/rows-after.txt`, then
   `diff ~/rows-before.txt ~/rows-after.txt` must print nothing. (When the release
   also brings migrations, the tables they change can differ; check those against the
   release notes.) Finally, sign in and open a training you have progress in: its media must load and it
   must resume where you left off. If you use imported SCORM courses, launch one too.

**Going back.** As long as the old volume exists: `docker compose down`, delete the
`volumes:` block from the override file (or, after an earlier upgrade, set its name back),
put back the previous `docker-compose.yml` and
`APP_VERSION`, and `docker compose up -d`. The old database comes back as it was at step 1.
Anything entered on the new version is lost.

**Removing the old volume.** Once the new version has run for a few days and has its own
backups:

```bash
docker volume rm afry_lms_postgres_data   # the name before this upgrade
```

Keep the `volumes:` block in the override file. It names the volume the database now
lives on, and removing it would start the database on an empty volume again. On the next
major version, change its name (`afry_lms_postgres_data_pg20`) and remove
`afry_lms_postgres_data_pg19` at the end.

**`pg_upgrade`** converts the data files in place instead of dumping and restoring them,
which only pays off for an unusually large database, where the restore would take hours. It
needs the old and the new PostgreSQL binaries in one container, which the `db` image does
not provide, and with `--link` the old data cannot be started again once the new server has
run. There is no runbook for it here: dump and restore is the supported procedure.

---

## Resource limits

Each container declares a CPU and memory limit under `deploy.resources.limits` in `docker-compose.yml`. These prevent a single misbehaving request — a large SCORM import, a slow Mistral API response, or heavy image processing via Sharp — from starving the other containers and taking down the whole host.

> **Requires Docker Compose v2** (the `docker compose` plugin). On the legacy `docker-compose` v1 CLI these limits are silently ignored. Verify with `docker compose version`.

The shipped defaults are conservative starting points suitable for a small deployment:

| Container | CPUs | Memory | Override in `.env` |
|---|--:|--:|---|
| `afry_lms_db` | 1.0 | 512M | `POSTGRES_CPUS` · `POSTGRES_MEMORY` |
| `afry_lms_backend` | 1.0 | 512M | `BACKEND_CPUS` · `BACKEND_MEMORY` |
| `afry_lms_frontend` | 0.25 | 64M | `FRONTEND_CPUS` · `FRONTEND_MEMORY` |
| `migrate` | 0.5 | 256M | `MIGRATE_CPUS` · `MIGRATE_MEMORY` |

**Calibrate before relying on them.** Run a representative load — concurrent learners plus a large SCORM import — and watch actual usage:

```bash
docker stats
```

If a container approaches its memory limit it will be OOM-killed (visible as `OOMKilled` in `docker compose ps` / `docker inspect`). Raise the relevant limit and restart.

Override them in `.env` rather than editing `docker-compose.yml`:

```env
BACKEND_MEMORY=1G
BACKEND_CPUS=2.0
POSTGRES_MEMORY=1G
```

Apply changes with `docker compose up -d` (recreates only the affected containers).

---

## Backup & recovery

A complete AFRY LMS backup is **two artifacts kept together**: an encrypted
`pg_dump` of the database **and** an encrypted archive of the uploaded files
(images and SCORM packages). A database-only backup is not recoverable on its own
— the database stores file UUIDs, and SCORM learner state lives only in the
database while the blobs live only in the `uploads_data` volume.

Encrypted daily backups of both, retention, scheduling, the tested restore
procedure, a backup health check, and RTO/RPO targets are documented in the
dedicated [Backup & Recovery runbook](./backup-and-recovery.md). Set up the
passphrase and `/srv/backups` as described there before going live.

---

## Rotating secrets

Every secret in `.env` can be replaced without signing anyone out or losing stored data.
Each step restarts the backend, so expect a few seconds of unavailability per run; pick a
quiet moment. `scripts/rotate-secret.sh` does it for you, and the manual steps are listed
below in case you prefer to do it by hand.

### When to rotate

| Situation | What to rotate |
|---|---|
| Once a year | Every secret in `.env`. Put it in the calendar next to the backup restore test |
| Someone who could read `.env`, the server or a backup leaves | Every secret in `.env`, plus the GHCR token if it was theirs |
| A secret may have leaked | Everything, now. See [If a secret has leaked](#if-a-secret-has-leaked) |
| Your email or AI provider's policy says so | The SMTP password or AI key, in the back office (below) |

Do not schedule password expiry for the people who use the product: current guidance
(NIST SP 800-63B) is to change a user's password when there is a reason to, not on a timer.
Password rules are set in the product, not here.

### How the overlap works

The backend reads each of `JWT_SECRET`, `FILE_TOKEN_SECRET`, `SCORM_CONTEXT_SECRET` and
`SMTP_ENCRYPTION_KEY` together with an optional `<NAME>_PREVIOUS`. It signs and encrypts
with the new value only, but still accepts the previous one. A rotation therefore has two
steps:

1. **Start.** The current value moves to `<NAME>_PREVIOUS` and a new value takes its place.
   Sessions, file links, SCORM launches and stored secrets created under the old value keep
   working.
2. **Finish.** Once nothing signed with the old value is still in use, remove
   `<NAME>_PREVIOUS`. From then on only the new value is accepted.

| Secret | What it protects | Finish after |
|---|---|---|
| `JWT_SECRET` | Sign-in sessions | The session lifetime under **System > Sign-In Policies** (12 hours by default, 30 days at most) |
| `FILE_TOKEN_SECRET` | Links to private files and documents, HTML package addresses, SCORM package access | 13 hours (an HTML package address lasts up to 13 hours) |
| `SCORM_CONTEXT_SECRET` | Running SCORM launches | 1 hour |
| `SMTP_ENCRYPTION_KEY` | Stored SMTP passwords and AI keys | Straight away, once re-encryption succeeds |

### With the script

Place `rotate-secret.sh` from the release assets in `/srv/afry_lms/scripts/` with the other
scripts. The host needs `openssl` and `curl`. Before you start:

- **Pin `APP_VERSION` in `.env`** to the release you are running. The script recreates the
  backend from whatever `APP_VERSION` says, and `latest` could swap in a newer image without
  its migrations.
- **If your deployment is not in `/srv/afry_lms`**, pass `COMPOSE_DIR` after `sudo`:
  `sudo COMPOSE_DIR=/opt/lms bash scripts/rotate-secret.sh ...`. Set before `sudo`, it is
  dropped. `HEALTH_TIMEOUT` (default 90 seconds) is how long the script waits for the backend.

```bash
cd /srv/afry_lms

# Sign-in sessions: start now, finish in a week
sudo bash scripts/rotate-secret.sh JWT_SECRET
sudo bash scripts/rotate-secret.sh JWT_SECRET --finish

# Same pattern for the two link-signing secrets (finish after 13 hours and 1 hour)
sudo bash scripts/rotate-secret.sh FILE_TOKEN_SECRET
sudo bash scripts/rotate-secret.sh SCORM_CONTEXT_SECRET

# Encryption key: rotates, then re-encrypts every stored SMTP password and AI key
sudo bash scripts/rotate-secret.sh SMTP_ENCRYPTION_KEY
sudo bash scripts/rotate-secret.sh SMTP_ENCRYPTION_KEY --finish

# Database password: changes the role and .env together
sudo bash scripts/rotate-secret.sh POSTGRES_PASSWORD
```

Each run:

- copies `.env` to `.env.bak.<timestamp>` (mode 600) before changing it. The copy holds the
  old secrets, so delete it once you are satisfied.
- recreates the backend container, then restarts the frontend container so its Nginx finds
  the new backend. The database keeps running.
- waits until `/api/version` reports a database version, which means the backend has
  started and reached the database. If it does not within `HEALTH_TIMEOUT`, the script
  restores `.env`, restarts the backend on the old values and tells you where to look.

For `SMTP_ENCRYPTION_KEY`, the script lists any stored secret that neither key can read and
leaves it as it is. That happens if a value was saved under an even older key, or is
damaged. Re-enter it in the back office, then finish the rotation. In this case `.env` is
not restored: both keys stay configured, so nothing stops working in the meantime.

For `POSTGRES_PASSWORD`, the script sends the new password to PostgreSQL over standard input,
so it never shows in a process listing. If the backend cannot connect afterwards, it sets the
role's password back as well as `.env`. If even that fails, it leaves both on the new
password, so they still agree, and tells you. The `postgres` container keeps the old value in its
environment until the next `docker compose up -d`. That is harmless: the image reads it only
when it creates a new, empty database.

### By hand

**For `JWT_SECRET`, `FILE_TOKEN_SECRET`, `SCORM_CONTEXT_SECRET` and `SMTP_ENCRYPTION_KEY`:**

1. In `.env`, add `<NAME>_PREVIOUS=` with the current value, and replace `<NAME>` with the
   output of `openssl rand -hex 32`.
2. Run `docker compose up -d --no-deps backend`, then `docker compose restart frontend`.
3. For `SMTP_ENCRYPTION_KEY` only, run
   `docker compose exec backend node packages/backend/src/db/reencrypt-secrets-cli.js`.
   It reports how many secrets it re-encrypted and exits non-zero if any could not be read.
4. After the wait in the table above, delete the `<NAME>_PREVIOUS` line and repeat step 2.

**For `POSTGRES_PASSWORD`:**

1. Generate a password: `openssl rand -hex 32`.
2. Change the role's password with psql's `\password` command. It prompts without echoing,
   and the password stays out of both your shell history and psql's. Use your
   `POSTGRES_USER` and `POSTGRES_DB` if you changed them:
   ```bash
   docker compose exec postgres psql -U afry_lms -d afry_lms_db
   # afry_lms_db=# \password afry_lms
   ```
3. Put the same value in `POSTGRES_PASSWORD` in `.env`.
4. Run `docker compose up -d --no-deps backend`, then `docker compose restart frontend`.
5. Check that `curl http://localhost:8081/api/version` shows a version in `"db"`. The
   endpoint answers even without a database, with `"db": null`, which means the password
   does not match.

Changing `.env` alone is not enough. PostgreSQL keeps the password it was given when the
volume was first created, so the backend can no longer connect.

### Secrets outside `.env`

| Secret | How to rotate it |
|---|---|
| SMTP password, system-wide | **System > Notifications**, SMTP section. A super-user enters the new password. Change it at the mail provider first |
| SMTP password, per organization | The organization's **Notification Settings**. An organization admin enters it |
| Mistral AI key, system-wide or per organization | **System > AI** or the organization's **AI Settings**. Create the new key at Mistral, choose **Set key**, then **Test key**, then revoke the old one at Mistral |
| GHCR token | Ask the AFRY LMS team for a new token, then run `docker login ghcr.io` again as in [Authenticate with GHCR](#4-authenticate-with-ghcr) |
| Backup passphrase | Write a new one to `/root/.backup-passphrase`. Keep the old passphrase for as long as the backups it encrypted are retained, or they cannot be restored. See the [Backup & Recovery runbook](./backup-and-recovery.md) |
| `SUPERADMIN_PASSWORD` | Used on first start only. The first sign-in with it must choose a new password before getting in. Remove it from `.env` afterwards |

The in-app ones are described for administrators in the manual's *Passwords and secrets*
article, at `/help/administration/passwords-and-secrets`.

### If a secret has leaked

Assume whoever has it can use it, and skip the overlap:

1. Rotate `JWT_SECRET`, `FILE_TOKEN_SECRET` and `SCORM_CONTEXT_SECRET`, and run each one's
   `--finish` straight away. Everyone is signed out, which is the point: a leaked
   `JWT_SECRET` lets anyone mint a session for any user.
2. Rotate `SMTP_ENCRYPTION_KEY` and finish it.
3. If the database or a backup may also have leaked, the stored SMTP passwords and AI keys
   are exposed too. Change them at the providers and enter the new ones in the back office.
4. Rotate `POSTGRES_PASSWORD`.
5. Ask super-users and administrators to change their passwords. An administrator can also
   send a reset email to any active user they manage, from the user's page. A completed
   reset ends every session that user had; changing your own password ends all your other
   sessions.
6. Review the audit trail under **System > Logs** for the period in question.
7. Replace the GHCR token, and start a new backup passphrase for future backups.

### Startup checks

In production, the backend refuses to start, and logs why, when:

- `JWT_SECRET`, `FILE_TOKEN_SECRET` or `SCORM_CONTEXT_SECRET` is missing, shorter than 32
  characters, or still the `CHANGE_ME` placeholder from `env.prod.example`
- `SMTP_ENCRYPTION_KEY` or `SMTP_ENCRYPTION_KEY_PREVIOUS` is set but is not 64 hex characters
- a `<NAME>_PREVIOUS` equals its `<NAME>`, or is set without it
- `POSTGRES_PASSWORD` is `secret` (the development default, which compose substitutes when
  `.env` sets none) or the placeholder

A value counts as the placeholder when it contains `CHANGE_ME`, in any case.

Read the reason with `docker compose logs backend`.

The backend, and before it the `migrate` one-shot, also refuses a database that a newer
release has migrated: see [Rolling back](#rolling-back). Under Compose, `migrate` stops first,
so read that one with `docker compose logs migrate`.

---

## Environment variables

Generate every secret with `openssl rand -hex 32`. Compose refuses to start
while `JWT_SECRET`, `FILE_TOKEN_SECRET` or `SCORM_CONTEXT_SECRET` is missing.

### Required

| Variable | What it is | Example |
|---|---|---|
| `GITHUB_REPOSITORY` | Image path in the registry, after the host | `acme/afry-lms` |
| `POSTGRES_PASSWORD` | Database password | strong random |
| `JWT_SECRET` | Signs sign-in tokens | secret |
| `FILE_TOKEN_SECRET` | Signs private-file URLs | secret |
| `SCORM_CONTEXT_SECRET` | Signs SCORM launch contexts | secret |
| `CORS_ORIGIN` | The public origin, exactly | `https://yourdomain.com` |
| `PORTAL_BASE_URL` | Base of the links in emails | `https://yourdomain.com` |
| `CONTENT_HOST` | Host name SCORM and HTML packages are served from. Empty runs them on the app's address, trusted: see [Content origin](#content-origin) | `content.yourdomain.com` |

### First run only

| Variable | What it is |
|---|---|
| `SUPERADMIN_EMAIL` | The first super-user. Ignored once one exists |
| `SUPERADMIN_PASSWORD` | Their first password, which must be changed at the first sign-in. Remove it from `.env` afterwards |

### Optional

| Variable | Default | What it does |
|---|---|---|
| `APP_VERSION` | `latest` | Image tag to run. Pin it, e.g. `v1.2.3` |
| `REGISTRY` | `ghcr.io` | Registry host the images are pulled from. See [Using your own registry](#using-your-own-registry) |
| `APP_PORT` | `8081` | Host port of the app, on `127.0.0.1`. Kept off 80 for the reverse proxy |
| `POSTGRES_USER` | `afry_lms` | Database user |
| `POSTGRES_DB` | `afry_lms_db` | Database name |
| `SMTP_ENCRYPTION_KEY` | none | Encrypts stored SMTP passwords and the AI key. Needed for email or AI ¹ |
| `JWT_SECRET_PREVIOUS`, `FILE_TOKEN_SECRET_PREVIOUS`, `SCORM_CONTEXT_SECRET_PREVIOUS`, `SMTP_ENCRYPTION_KEY_PREVIOUS` | none | The old value during a rotation. Set and removed by `rotate-secret.sh`. See [Rotating secrets](#rotating-secrets) |
| `EXTERNAL_FONTS` | `bunny` | `off` keeps font pickers to web-safe fonts. See [Air-gapped installs](#air-gapped-and-no-egress-installs) |
| `DEFAULT_STORAGE_QUOTA_BYTES` | unlimited | Initial value of the default storage quota for organizations without their own. Read once ² |
| `TRUST_PROXY` | `2` | Number of proxies in front of the backend ³ |
| `LOG_RETENTION_MIN_DAYS` | `30` | Shortest log retention window a super-user may set ⁴ |
| `MANUAL_PURGE_ENABLED` | `true` | `false` removes **Purge all now** from **System > Data Lifecycle Policies** |
| `SESSION_LIFETIME_HOURS` | `12` | How long a sign-in lasts, 1 to 720 hours. Default for **System > Sign-In Policies > Sessions**, where a value set wins |
| `SESSION_IDLE_MINUTES` | `60` | Minutes without activity before someone is signed out. `0` turns it off, otherwise 5 to 1440. Default for the same page |
| `*_CPUS`, `*_MEMORY` | see table | Container limits. See [Resource limits](#resource-limits) |

¹ Rotate it with `scripts/rotate-secret.sh SMTP_ENCRYPTION_KEY`, which keeps stored SMTP
passwords and AI keys readable and re-encrypts them. Replacing the value in `.env` by hand
without `SMTP_ENCRYPTION_KEY_PREVIOUS` makes every stored one unreadable. See
[Rotating secrets](#rotating-secrets).

² In bytes, e.g. `10737418240` for 10 GB. Empty or `0` means unlimited. The backend copies
it into **System > Storage & Upload Limits** on its first start (on a new install, or on
the first start of a release that has that field) and never reads it again, so a later
change to `.env` has no effect: change the quota on that page instead. A quota set on the
organization (**Storage & Upload Limits** in its **More actions** menu) always wins.

³ Count the frontend container's own Nginx plus every proxy in front of it. `2`
fits Caddy, Nginx or Application Gateway. Add one per extra layer, e.g. `3` for
Azure Front Door ahead of Application Gateway. The backend uses it to find the
user's IP for the login log and the rate limits. Too low and everyone looks like a
proxy; too high and a client can forge its own IP. Anything but a whole number is
logged and ignored, and the backend then trusts one hop.

⁴ Applies to both the login log and the audit trail under **System > Data
Lifecycle Policies**. A window already stored below it is treated as this value.
Anything but a whole number of 1 or more falls back to `30`.

---

## Verifying the images

Every release publishes these files next to the images:

| File | What it is |
|---|---|
| `digests.txt` | The digest of each image, one `image@sha256:…` per line. A tag such as `v1.2.3` can be moved to another image; a digest names one exact image and cannot. The release notes list them too |
| `cosign.pub` | The AFRY LMS public key the images are signed with |
| `db.cdx.json`, `backend.cdx.json`, `frontend.cdx.json` | One SBOM per image (see [What the SBOMs are for](#what-the-sboms-are-for)) |

### Checking the signatures

Install [cosign](https://github.com/sigstore/cosign/releases) 3 or later on any machine that
can pull the images. It is a single binary, and the server itself does not need it. The
images are private, so log in with your pull token first, as in
[step 4](#4-authenticate-with-ghcr); cosign uses the same login as Docker. Then, in a directory
holding the release's `digests.txt` and `cosign.pub`:

```bash
while read -r ref; do
  cosign verify --key cosign.pub --insecure-ignore-tlog "$ref"
done < digests.txt
```

Every image must report `The signatures were verified against the specified public key`. Any
error means the image is not the one AFRY LMS released: do not run it, and contact the
AFRY LMS team.

The images are signed with a key, not with Sigstore's keyless signing, and the signatures are
deliberately not recorded in Sigstore's public transparency log. `--insecure-ignore-tlog`
tells cosign not to look for them there, so the warning it prints about skipping the log is
expected. Nothing is checked against Sigstore's services: the check needs the registry and
nothing else.

To check what a tag points at today, pass the tag instead of the digest:
`cosign verify --key cosign.pub --insecure-ignore-tlog ghcr.io/xavier-follet/afry-lms/backend:v1.2.3`.

### Checking without internet access

When the machine that must do the check cannot reach the registry, save the images with
their signatures on a connected machine:

```bash
while read -r ref; do
  name=$(basename "${ref%@*}")   # db, backend or frontend
  cosign save --dir "$name" "$ref"
done < digests.txt
```

Move the three directories and `cosign.pub` across, then check them where they landed. This
needs no network at all:

```bash
for name in db backend frontend; do
  cosign verify --key cosign.pub --insecure-ignore-tlog --local-image "$name"
done
```

The same directories load into a registry of your own with `cosign load`, signatures
included: see [Using your own registry](#using-your-own-registry).

### Pinning by digest

`docker-compose.yml` pulls by tag (`APP_VERSION`). To run exactly the images you verified,
pin each service to its digest in a `docker-compose.override.yml` next to it. Compose reads
that file on its own:

```yaml
services:
  postgres:
    image: ghcr.io/xavier-follet/afry-lms/db@sha256:<db digest>
  migrate:
    image: ghcr.io/xavier-follet/afry-lms/backend@sha256:<backend digest>
  backend:
    image: ghcr.io/xavier-follet/afry-lms/backend@sha256:<backend digest>
  frontend:
    image: ghcr.io/xavier-follet/afry-lms/frontend@sha256:<frontend digest>
```

Copy the digests from `digests.txt`. With your own registry, replace `ghcr.io` with its host;
the digests stay the same. Update the file on every upgrade: while it exists, `docker compose
pull` keeps fetching the pinned images whatever `APP_VERSION` says.

### What the SBOMs are for

Each SBOM (software bill of materials) lists the operating-system and application packages
inside one image, with their versions and licences, in CycloneDX JSON. It describes the
`linux/amd64` image; the `linux/arm64` image is built from the same Dockerfile. Use them to
know what you are running, to feed your own vulnerability scanner (for example
`trivy sbom backend.cdx.json`), or for a licence review.

An SBOM is an inventory, not a verdict: it does not say an image is free of vulnerabilities.
Each release is scanned before it is published, and a HIGH or CRITICAL finding with a fix
available stops it, but new vulnerabilities are found in old packages every day.

### When the signing key changes

- The private key is held by the AFRY LMS release team: in the release pipeline's secret
  store, with one offline backup. It is not used anywhere else.
- Every release attaches the public key it was signed with, and the current one is
  `cosign.pub` at the root of the [releases repository](https://github.com/xavier-follet/afry-lms-releases).
  Keep the copy you received at installation.
- The key changes only on a planned rotation or if it may have been exposed. The AFRY LMS
  team tells you through your usual contact before the first release signed with a new key,
  and that release says so at the top of its release notes.
- If a release's `cosign.pub` differs from the key you have and you were not told, do not
  install that release; contact the AFRY LMS team.
- Older releases remain verifiable with the `cosign.pub` attached to each of them.

---

## Using your own registry

When the server cannot reach `ghcr.io` (no internet after installation, or a policy that
allows only an internal registry such as Azure Container Registry), copy the images into a
registry it can reach and point Compose at it. Images are pulled from
`$REGISTRY/$GITHUB_REPOSITORY/<db|backend|frontend>:$APP_VERSION`.

1. On a machine with [cosign](https://github.com/sigstore/cosign/releases) 3 or later, logged
   in to both registries, copy the three images. Keep the repository path and the tag:
   ```bash
   SRC=ghcr.io/xavier-follet/afry-lms
   DST=myregistry.azurecr.io/xavier-follet/afry-lms
   for img in db backend frontend; do
     cosign save --dir "$img" "$SRC/$img:v1.2.3"
     cosign load --dir "$img" "$DST/$img:v1.2.3"
   done
   ```
   This copies both architectures and the signatures, and keeps the digests, so
   `digests.txt` still applies and the [signature check](#checking-the-signatures) works
   against your registry. Do not copy with `docker pull`, `docker tag` and `docker push`:
   that copies only your machine's architecture, gives the image a new digest, and leaves
   the signature behind. `cosign copy` is deprecated and does not copy the signatures either.
   When no machine reaches both registries, run `cosign save` on one side and `cosign load`
   on the other.
2. In `.env` on the server, set the registry host and pin the version:
   ```env
   REGISTRY=myregistry.azurecr.io
   APP_VERSION=v1.2.3
   ```
   If your registry keeps the images under a different path, set `GITHUB_REPOSITORY` to that
   path as well.
3. Run `docker login myregistry.azurecr.io` on the server instead of logging in to `ghcr.io`,
   then `docker compose pull` and `docker compose up -d`.

The backup scripts use one more image, `alpine`, from Docker Hub. Copy it too (with `docker buildx imagetools create`) and
set `BACKUP_HELPER_IMAGE`: see [The helper image](./backup-and-recovery.md#the-helper-image).

Repeat step 1 for every release before [updating](#updating-the-application). Use `REGISTRY`
rather than setting `image:` in `docker-compose.override.yml`: it is one line, and it applies
to all four services. The exception is [pinning by digest](#pinning-by-digest), which needs
the override file anyway.

---

## Air-gapped and no-egress installs

**A stock install makes no request to any external host.** Every font the product uses for
itself — the back-office chrome and all nine portal template families — is served from your
own container. There is nothing to configure for this and no way to turn it off.

One thing remains open-ended: a font a **course author** picks for a training theme, or an
**organisation admin** picks for a portal, can be any family in existence, so it cannot be
shipped in advance. Those are fetched from `fonts.bunny.net` (a privacy-friendly Google
Fonts mirror — no cookies, no logging, no request to Google) when someone chooses one.

If this host has no outbound internet access, or your policy forbids third-party requests,
set:

```bash
EXTERNAL_FONTS=off
```

and restart the backend. The font pickers in the theme editor and in **Organizations >
Design** then offer web-safe families only — Arial, Georgia, Courier New and the like, which
every browser already has. Any web font already saved in a theme keeps its name but falls
back to the next family in its stack.

Verify it the way you would verify anything else: open the admin, the editor, a portal and a
player page with the browser's network tab filtered to third-party hosts. The list should be
empty.

### What the flag does not cover

**Imported SCORM packages.** A package authored elsewhere brings its own HTML and CSS, and
many of them open with a `@import url('https://fonts.googleapis.com/…')`. Those files are
served from your own origin under `/files/private/`, but the stylesheet inside them still
tells the learner's browser to fetch from Google — and no setting of ours can rewrite
somebody else's course. `EXTERNAL_FONTS` governs the platform; it cannot govern content you
imported into it.

If your site must make no third-party request at all, treat SCORM imports as content to vet
before uploading, the same way you would vet an embedded video. To enforce it rather than
trust it, put a `Content-Security-Policy: font-src 'self'` header on the `/files/` location
in your reverse proxy — the browser then refuses the request whatever the package asks for.

This is a per-installation decision, not a per-customer one, which is why it is an
environment variable and not a setting in the UI. See [docs/font-policy.md](../font-policy.md)
for the reasoning.

### The backup helper image

`backup.sh` and `restore.sh` need one image the stack itself does not run: a pinned `alpine`
from Docker Hub. On a host without internet, `docker load` it from a machine that has
internet and name it in `BACKUP_HELPER_IMAGE`, as described in
[The helper image](./backup-and-recovery.md#the-helper-image). Do not run
`docker image prune -a` or `docker system prune -a` on the host afterwards: they delete it,
and the next backup then stops with exit code `2`.

---

## Ports reference

| Container | Port | Reachable from | Serves |
|---|--:|---|---|
| `afry_lms_frontend` | 8081 | `127.0.0.1` only (`APP_PORT`) | The SPAs and uploaded files. Proxies `/api/` to the backend, and on `CONTENT_HOST` only package files |
| `afry_lms_backend` | 3200 | Docker network only | The REST API |
| `afry_lms_db` | 5432 | Docker network only | PostgreSQL |

With Application Gateway, the frontend is also bound to the VM's private IP. See
[Option C](#option-c--azure-application-gateway).

---

## Troubleshooting

**Container won't start**
```bash
docker compose logs backend
docker compose logs frontend
docker compose logs postgres
```

**DB connection refused**
`POSTGRES_HOST` is not a variable you set — it is hardcoded to `postgres` (the Docker service name) inside the compose file.

**CORS errors in browser**
`CORS_ORIGIN` must match the exact origin the browser uses, including scheme and port.

**Frontend shows a blank page**
If the `VITE_API_URL` repository secret is set, the frontend image is built with that API URL baked in at CI time, and a domain change needs a new release. If it is empty, the frontend calls `/api` on whatever domain served it and needs no rebuild. In that case check `docker compose logs frontend` and the browser console instead.

**Pull access denied**
Re-authenticate: `echo YOUR_TOKEN | docker login ghcr.io -u afry-lms-customer --password-stdin`. If it still fails, the token may have expired or been revoked: ask the AFRY LMS team for a new one.
