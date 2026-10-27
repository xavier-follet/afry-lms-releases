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
```

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
(`< 26 h`), is non-empty, and **decrypts** with the stored passphrase (catching a
silently rotated passphrase or a truncated file). It prints one line and exits
non-zero with a distinct code per failure so cron or an external monitor can alert.

```bash
sudo /srv/afry_lms/scripts/backup-healthcheck.sh
# OK: backup 20260622-030000 — 2h old, db 184320B, files+db present and decryptable
```

## 9. Restore

> **Destructive.** Restore overwrites the live database and uploads volume. The
> script asks for confirmation unless you pass `--yes`.

Assumes the running stack is the **same app version** the backup was taken from
(same-version restore needs no migrations). For a cross-version restore, the final
`docker compose up -d` runs the `migrate` one-shot, which applies any pending
migrations.

```bash
# Full restore of the newest backup
sudo /srv/afry_lms/scripts/restore.sh latest

# A specific point in time
sudo /srv/afry_lms/scripts/restore.sh 20260622-030000

# Partial restores
sudo /srv/afry_lms/scripts/restore.sh latest --db-only
sudo /srv/afry_lms/scripts/restore.sh latest --files-only
```

What it does: decrypt-tests both halves → stops `frontend` and `backend` (both
mount the uploads volume; backend writes the DB) → clears and re-extracts the
uploads volume → pipes the dump into `psql` (the dump is `--clean --if-exists`, so
it is idempotent) → `docker compose up -d` → smoke-checks the stack. Afterwards,
**spot-check a known image and launch a SCORM course** to confirm files and learner
state came back together.

### Recovering on a fresh host (host loss)

1. Provision the host, install Docker, set up
   [encryption at rest](./production-deployment.md#encryption-at-rest).
2. Follow [first-time setup](./production-deployment.md#first-time-setup): create
   `/srv/afry_lms`, download the release files (incl. `scripts/`), restore `.env`,
   `docker compose pull`, then `docker compose up -d` once to create the volumes.
3. Copy the backup triple (from offsite) into `/srv/backups` and restore the
   passphrase to `/root/.backup-passphrase`.
4. `sudo /srv/afry_lms/scripts/restore.sh latest --yes`.

## 10. Test-your-restore drill

A backup you have never restored is a hypothesis. **Quarterly**, restore into a
throwaway stack instead of production:

```bash
# Separate project name + backup dir so prod is never touched
COMPOSE_PROJECT_NAME=afry_lms_drill \
COMPOSE_DIR=/srv/afry_lms \
BACKUP_DIR=/srv/backups \
  /srv/afry_lms/scripts/restore.sh latest --yes
```

Verify row counts match, a known image loads, and a SCORM course launches with
prior progress intact. Tear the drill stack down afterwards.

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
| `uploads volume '…' not found` | Auto-detection failed; set `UPLOADS_VOLUME` to the host volume name (`docker volume ls`). |
| `cannot decrypt …` in healthcheck | Passphrase at `/root/.backup-passphrase` no longer matches the one used to create the backup, or the file is truncated. |
| Restored images 404 but data is present | A files-only or db-only restore was run, or the two halves came from different timestamps. Restore a matched pair. |
| Offsite copy keeps failing | Backups still succeed locally; fix the `OFFSITE_DEST` transport (SSH key, path, permissions). |
