# Backup & Recovery

Operational runbook for backing up and restoring an AFRY LMS deployment. This is
the single source of truth for disaster recovery — keep a copy somewhere you can
reach when the server is down.

## 1. Overview & scope

A complete AFRY LMS backup is **two artifacts that must be kept together**:

| Artifact | Contains | Lives in |
|---|---|---|
| Database dump | All users, organizations, trainings, structure, **SCORM learner runtime state** (scores, completion, suspend data), and **file metadata** (the UUIDs) | PostgreSQL (`postgres_data` volume) |
| Files archive | The actual uploaded blobs: images, videos, audio, and unpacked **SCORM packages** | `uploads_data` volume |

Neither is usable alone. The database stores file UUIDs that reference blobs in
the volume, so a **database-only restore yields broken images and missing SCORM
packages**, and a **files-only restore loses all learner progress**. Always back
up and restore both.

Both artifacts are **GPG (AES256) encrypted** — they contain PII in plaintext and
must never be stored or shipped offsite unencrypted. This is data-in-transit/at-rest
protection for the *backup files*; it is separate from the host-disk
[encryption at rest](./production-deployment.md#encryption-at-rest) that protects
the live volumes.

## 2. What gets backed up & where

The `scripts/backup.sh` script produces, per run (`<TS>` = `YYYYMMDD-HHMMSS`):

```
/srv/backups/afry_lms-files-<TS>.tar.gz.gpg   # uploads volume
/srv/backups/afry_lms-db-<TS>.sql.gpg         # pg_dump --clean --if-exists
/srv/backups/afry_lms-<TS>.manifest           # pairs the two: sizes, sha256, app image
```

## 3. One-time setup

```bash
# 1. Backup passphrase, readable only by root. Store a copy in your password
#    manager — if you lose it, every backup is unrecoverable.
echo "your-strong-backup-passphrase" | sudo tee /root/.backup-passphrase
sudo chmod 600 /root/.backup-passphrase

# 2. Backup destination
sudo mkdir -p /srv/backups

# 3. Make the scripts executable (they ship with the release alongside docker-compose.yml)
sudo chmod +x /srv/afry_lms/scripts/*.sh

# 4. Pull the helper image the scripts copy the uploads volume with (see below)
sudo docker pull "$(grep -o 'alpine:[0-9.]*@sha256:[0-9a-f]*' /srv/afry_lms/scripts/backup.sh)"
```

### The helper image

`backup.sh` and `restore.sh` read and write the uploads volume through a small `alpine`
container. The image is pinned by digest in both scripts (`BACKUP_HELPER_IMAGE`), so it is
the same image on every host and every run, and each release can move it on. Before using
it, both scripts check that it is present. When it is not, they pull it, and when the pull
fails they stop before touching anything (`backup.sh` exits `2`, `restore.sh` exits `5`).

Pull it once during installation, as above, so a backup does not depend on Docker Hub being
reachable at 03:00. After that:

- **Never `docker image prune -a` or `docker system prune -a` on this host.** `-a` removes
  every image no container is using, and the helper image is used only for the seconds a
  backup runs. Plain `docker image prune` (dangling images only) is safe. If you prune with
  `-a` anyway, pull the image again.
- **Mirror or internal registry:** set `BACKUP_HELPER_IMAGE` in `/srv/afry_lms/.env` to the
  same image in your registry. Copy it with `docker buildx imagetools create --tag
  myregistry.example.com/library/alpine:3.24 alpine:3.24@sha256:<digest>`, which keeps both
  architectures and the digest, so you can keep the `@sha256:` part.
- **No internet on the host:** on a machine that has it, pull the image for the server's
  architecture, give it a tag, and save it. `docker save` does not keep the digest, so
  name the tag in `.env` instead:

  ```bash
  # On a machine with internet (use linux/arm64 for an ARM server)
  IMG=$(grep -o 'alpine:[0-9.]*@sha256:[0-9a-f]*' scripts/backup.sh)
  docker pull --platform linux/amd64 "$IMG"
  docker tag "$IMG" alpine:afry-backup-helper
  docker save --platform linux/amd64 -o backup-helper.tar alpine:afry-backup-helper

  # On the server
  sudo docker load -i backup-helper.tar
  echo 'BACKUP_HELPER_IMAGE=alpine:afry-backup-helper' | sudo tee -a /srv/afry_lms/.env
  ```

  Repeat this when a release changes the digest in the scripts.

Optional: set an **offsite destination** so each backup is copied off the host.
A backup that only lives on the same machine does not survive host loss.

```bash
# Append to /srv/afry_lms/.env (any rsync-reachable target)
OFFSITE_DEST=backups@offsite.example.com:/afry_lms
```

## 4. Running a backup

```bash
sudo /srv/afry_lms/scripts/backup.sh
```

Configurable via environment **or via `/srv/afry_lms/.env`** (the scripts load it,
so these work under cron too; an explicit environment variable still wins). Defaults shown:

| Variable | Default | Purpose |
|---|---|---|
| `COMPOSE_DIR` | `/srv/afry_lms` | Deployment dir (holds `docker-compose.yml` + `.env`) |
| `BACKUP_DIR` | `/srv/backups` | Where encrypted backups are written |
| `PASSPHRASE_FILE` | `/root/.backup-passphrase` | GPG symmetric passphrase |
| `KEEP_DAILY` / `KEEP_WEEKLY` | `7` / `4` | Retention (see §6) |
| `OFFSITE_DEST` | _(empty)_ | rsync target; empty disables offsite |
| `UPLOADS_VOLUME` | _auto-detected_ | Override only if auto-detection picks the wrong volume |

The uploads volume name is **project-prefixed** on the host (e.g.
`afry_lms_uploads_data`). The script auto-detects it from the Docker Compose
labels; set `UPLOADS_VOLUME` only if you run several AFRY LMS stacks on one host.

A successful run ends with the log line `backup <timestamp> complete`. Every line also
goes to `$BACKUP_DIR/backup.log`.

| Exit code | Meaning |
|---|---|
| `0` | Backup written. A failed offsite copy still exits `0`; it logs a `WARNING` instead |
| `2` | Preflight failed: backup dir not writable, passphrase file missing, Docker unreachable, [helper image](#the-helper-image) missing and not pullable, PostgreSQL not ready, or uploads volume not found |
| `3` | File backup failed |
| `4` | Database backup failed |
| `5` | The file or database backup came out empty |

Any other non-zero code is an unexpected command failure; check the log.

## 5. Ordering & consistency

`backup.sh` captures **files first, then the database** — deliberately. The
database references file UUIDs, so taking the file archive first guarantees every
UUID the later dump can mention already exists in the file backup. A file uploaded
in the gap between the two steps appears in the file archive but not the dump — a
harmless orphan that the app's normal 48 h orphan cleanup would remove anyway.
The reverse order could record a DB row pointing at a blob missing from the file
backup, producing a broken restore. Blobs are immutable and UUID-addressed, so no
database lock or downtime is needed during backup.

## 6. Retention & offsite

`backup.sh` keeps the last `KEEP_DAILY` daily backups plus one per ISO-week for
`KEEP_WEEKLY` weeks; older triples (`files` + `db` + `manifest`) are pruned
together. A backup is never half-deleted.

If `OFFSITE_DEST` is set, the new triple is `rsync`'d offsite after each run.
Offsite failure **warns but does not fail** the backup — the local copy is still
valid — so fix the transport and the next run re-syncs.

## 7. Scheduling

Run a daily encrypted backup and a health check via cron (`sudo crontab -e`):

```cron
MAILTO=ops@yourdomain.com

# Daily backup at 03:00
0 3 * * * /srv/afry_lms/scripts/backup.sh >> /srv/backups/backup.log 2>&1

# Health check at 04:30 (after the backup has finished)
30 4 * * * /srv/afry_lms/scripts/backup-healthcheck.sh || echo "AFRY LMS backup healthcheck FAILED" | mail -s "AFRY LMS DR" ops@yourdomain.com
```

Cron runs with a bare environment — the scripts use absolute paths and default
`COMPOSE_DIR=/srv/afry_lms`, so no extra setup is needed.

## 8. Health check

`scripts/backup-healthcheck.sh` verifies the newest backup pair exists, is recent
(`< 26 h`), is non-empty, and **decrypts in full** with the stored passphrase
(catching a silently rotated passphrase or a truncated file). It does not restore
anything; [verifying a backup](#10-verifying-a-backup-restore-drill) does. It prints one line and exits
non-zero with a distinct code per failure so cron or an external monitor can alert.

```bash
sudo /srv/afry_lms/scripts/backup-healthcheck.sh
# OK: backup 20260622-030000 — 2h old, db 184320B, files+db present and decryptable
```

## 9. Restore

> **Destructive.** Restore overwrites the live database and uploads volume. The
> script asks for confirmation unless you pass `--yes`.

Set `APP_VERSION` in `.env` to the version the backup was taken on before restoring. The
manifest records it on its `app_image=` line. When the two differ:

- **Backup older than the images:** the final `docker compose up -d` runs the `migrate`
  one-shot, which applies the migrations the backup is missing. The site comes up on the
  newer version.
- **Backup newer than the images** (a host rebuilt on an older release, say): the dump
  restores, then `migrate` refuses with `Refusing to start: the database was migrated by a
  newer release` and exits 1, and the backend stays down. Set `APP_VERSION` to the
  `app_image=` version and `docker compose up -d`. Older code never runs on a newer schema.

The database is replaced whole, in one transaction: the `public` schema is emptied, then the
dump is loaded. A rollback leaves nothing a newer release created behind, and a restore that
fails partway leaves the database exactly as it was. Rolling back an upgrade is this restore
plus the older `APP_VERSION`: see
[Rolling back](./production-deployment.md#rolling-back). The same `--db-only` restore moves the database to a new PostgreSQL major
version: see [PostgreSQL major-version upgrades](./production-deployment.md#postgresql-major-version-upgrades).

```bash
# Full restore of the newest backup
sudo /srv/afry_lms/scripts/restore.sh latest

# A specific point in time
sudo /srv/afry_lms/scripts/restore.sh 20260622-030000

# Partial restores
sudo /srv/afry_lms/scripts/restore.sh latest --db-only
sudo /srv/afry_lms/scripts/restore.sh latest --files-only
```

| Exit code | Meaning |
|---|---|
| `0` | Restored, and the stack was started again |
| `2` | Bad arguments, or you did not type `YES` |
| `3` | Backup file missing |
| `4` | A backup file does not decrypt |
| `5` | Docker unreachable, or the [helper image](#the-helper-image) is missing and cannot be pulled. Nothing was stopped |

What it does: decrypts both halves in full → stops `frontend` and `backend` (both
mount the uploads volume; backend writes the DB) → clears and re-extracts the
uploads volume → empties the `public` schema and loads the dump into `psql`, in one
transaction → `docker compose up -d` → smoke-checks the stack. Afterwards,
**open a training you have progress in** and check that its images, videos and
attachments load and that it resumes where you left off. That confirms files and learner
state came back together. If you use imported SCORM courses, launch one too.

### Recovering on a fresh host (host loss)

1. Provision the host, install Docker, set up
   [encryption at rest](./production-deployment.md#encryption-at-rest).
2. Follow [first-time setup](./production-deployment.md#first-time-setup): create
   `/srv/afry_lms`, download the release files (incl. `scripts/`), restore `.env`,
   `docker compose pull`, then `docker compose up -d` once to create the volumes.
3. Copy the backup triple (from offsite) into `/srv/backups` and restore the
   passphrase to `/root/.backup-passphrase`.
4. `sudo /srv/afry_lms/scripts/restore.sh latest --yes`.

## 10. Verifying a backup (restore drill)

A backup you have never restored is a hypothesis. `scripts/verify-backup.sh` proves
one restores **without touching live data**, so it is safe to run on the production
host at any time. Run it **quarterly** at least, and after any change to the stack.

```bash
sudo /srv/afry_lms/scripts/verify-backup.sh latest
sudo /srv/afry_lms/scripts/verify-backup.sh 20260622-030000
# OK: backup 20260622-030000 — checksums match, decrypts in full, 412 archive entries, 87 tables, 1520 users, 64 trainings, last migration 0053_system_message.sql
```

It stops at the first failed check:

1. Both files match the sha256 recorded in the manifest.
2. Both files decrypt in full.
3. The uploads archive lists cleanly (`tar -tz`; nothing is extracted).
4. The database dump is restored into a **throwaway PostgreSQL container** started
   from the stack's `postgres` image, with `ON_ERROR_STOP`, no network and no named
   volume. It must contain tables, and `organizations` and `users` must not be
   empty. The container and its data are removed when the script exits, whether
   it passed or failed.

It never connects to the live database: the dump is written with
`--clean --if-exists`, so piping it into the wrong database would drop that
database's objects. The script refuses to restore into any container that belongs
to a Compose project or has a network.

**Resources.** The throwaway container needs about **256 MB of free memory** and
free disk under Docker's data root (`/var/lib/docker`) of roughly **three times the
decrypted dump size** (the restored tables plus their indexes). Nothing is written
to the backup directory or the host's temp space; decryption is streamed.

Configuration comes from the same environment or `.env` as the other scripts
(`BACKUP_DIR`, `PASSPHRASE_FILE`, `PG_USER`, `PG_DB`), plus:

| Variable | Default | Purpose |
|---|---|---|
| `DB_IMAGE` | _read from `docker-compose.yml`_ | Image for the throwaway container. Set it when verifying on a host without the deployment files. |
| `READY_TIMEOUT` | `120` | Seconds to wait for the throwaway PostgreSQL to accept connections. |

| Exit code | Meaning |
|---|---|
| `0` | Backup verified |
| `2` | Bad arguments |
| `3` | Backup file, manifest or passphrase file missing |
| `4` | A file's sha256 does not match the manifest (corrupted or replaced since backup) |
| `5` | A file does not decrypt in full (wrong passphrase, or truncated) |
| `6` | The uploads archive is not a readable `tar.gz` |
| `7` | Docker unreachable, image unknown, or the throwaway container did not start |
| `8` | The dump did not restore (the `psql` error is printed above) |
| `9` | The restored database fails a sanity check |

The script proves the backup restores at the database level. It does not run
migrations, so it does not prove the schema is current for a newer app version;
the last applied migration is printed so you can compare it with the release. To
also check files against learner progress, restore onto a spare host
(see [Recovering on a fresh host](#recovering-on-a-fresh-host-host-loss)) and
open a training with prior progress and check its media loads.

## 11. RTO / RPO

| Target | Value | Rationale |
|---|---|---|
| **RPO** (max data loss) | **24 h** | Daily 03:00 backup. Worst case loses up to one day of activity. |
| **RTO** (time to recover) | **2–4 h** | Fresh host: provision → `docker compose pull` → restore files (minutes) → restore DB (minutes at current data sizes) → `up -d`. The band absorbs human response time and fetching the offsite copy. |

A same-host recovery (data corruption, not host loss) is faster — roughly
**30–60 min**, since the images and volumes already exist and only `restore.sh`
runs.

These targets suit the current pre-release stage. **Revisit at GA / first paying
tenant** — a paying B2B customer will likely require a tighter RPO (see §13).

## 12. Appendix A — Encryption at rest

Backup *files* are GPG-encrypted by these scripts. The live *volumes* on the host
are protected separately by full-disk encryption — see
[Encryption at rest](./production-deployment.md#encryption-at-rest) in the
production deployment guide. Both layers are needed: one protects copies you ship
offsite, the other protects the running server's disk.

## 13. Appendix B — Point-in-time recovery (optional, not enabled)

Daily `pg_dump` gives a 24 h RPO. To drop the database RPO toward minutes you
would enable PostgreSQL **WAL archiving / PITR**:

- `archive_mode = on` + an `archive_command` (or `pg_receivewal`) shipping WAL
  segments continuously to offsite storage;
- periodic `pg_basebackup` base backups;
- monitoring of archive lag;
- recovery via `restore_command` + `recovery_target_time`.

This adds always-on moving parts that conflict with the simple, manual,
image-based deployment posture and is **not warranted before there is paying-customer
data**. Note also that **file backups have no PITR equivalent** here: even with
DB PITR, the uploads volume RPO would still be bounded by the daily file archive,
so DB-only PITR has limited end-to-end value until file backup frequency also
rises. Reconsider when the first production tenant onboards or a contractual RPO
below 24 h is required.

## 14. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `postgres is not ready` on backup | Stack is down — `docker compose up -d` first. |
| `helper image … is missing and cannot be pulled` | The image was pruned, or the host has no internet. Pull or load it: see [The helper image](#the-helper-image). |
| `uploads volume '…' not found` | Auto-detection failed; set `UPLOADS_VOLUME` to the host volume name (`docker volume ls`). |
| `cannot decrypt …` in healthcheck | Passphrase at `/root/.backup-passphrase` no longer matches the one used to create the backup, or the file is truncated. |
| `verify-backup.sh` exits `4` | The file changed after the manifest was written: a damaged disk, a partial offsite copy, or a file from another backup renamed into place. Use a different backup and check the storage. |
| `verify-backup.sh` exits `8` | The dump is incomplete or does not load into this PostgreSQL version. Read the `psql` error printed above the failure; never restore that backup over live data. |
| Restored images 404 but data is present | A files-only or db-only restore was run, or the two halves came from different timestamps. Restore a matched pair. |
| Offsite copy keeps failing | Backups still succeed locally; fix the `OFFSITE_DEST` transport (SSH key, path, permissions). |
