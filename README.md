# Immich Docker Setup

A Docker Compose setup for [Immich](https://immich.app) — a self-hosted photo and
video management solution. The `compose.yaml` is taken verbatim from the official
release so it stays in sync with upstream:
<https://docs.immich.app/install/docker-compose>

## Services

- **immich-server** — API + web UI (port `2283`)
- **immich-machine-learning** — ML features (search, face recognition, smart albums)
- **redis** — Valkey, used for job queues and caching
- **database** — PostgreSQL with the VectorChord/pgvecto extensions

> **Note:** Immich requires its own bundled Postgres image
> (`ghcr.io/immich-app/postgres:...`). It ships the vector-search extensions Immich
> depends on, so it cannot be replaced with a plain Postgres or a shared/global
> database instance.

## Prerequisites

- Docker
- Docker Compose v2

## Setup

### 1. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and update:

**Required:**
- `UPLOAD_LOCATION` — host path for your photo/video library
  (default: `/volume1/docker/immich/library`)
- `DB_DATA_LOCATION` — host path for the Postgres data
  (default: `/volume1/docker/immich/postgres`) — **must be local storage, not a network share**
- `DB_PASSWORD` — a secure random password (use only `A-Za-z0-9`)

**Optional:**
- `TZ` — timezone (default: `America/Chicago`)
- `IMMICH_VERSION` — `v3` tracks the latest v3.x release (upstream default); pin to `v3.0.1` for zero surprises. Avoid `release` — it would auto-jump to a future v4 with breaking changes.

### 2. Create the data directories

The compose file mounts host subdirectories under each parent (a small divergence from
upstream — the parent is kept for READMEs / backup scripts, and the container sees the
child as its data root):

- `UPLOAD_LOCATION/data` → `/data` inside `immich-server`
- `DB_DATA_LOCATION/db` → `/var/lib/postgresql/data` inside `immich_postgres`

```bash
mkdir -p /volume1/docker/immich/library/data
mkdir -p /volume1/docker/immich/postgres/db
```

### 3. Start the stack

```bash
docker compose up -d
```

### 4. Access Immich

Open `http://<host>:2283` and create your admin account on first launch.

## Maintenance

```bash
# View logs
docker compose logs -f

# Update to the latest release
docker compose pull
docker compose up -d

# Stop
docker compose down
```

## Synology RS1221+ notes

This stack is targeted at a Synology RS1221+.

- **CPU** — AMD Ryzen V1500B has no integrated GPU and no video encode/decode block,
  so hardware transcoding and ML acceleration are **not available**. The stack runs
  CPU-only; the `extends:` blocks in `compose.yaml` should stay commented out.
- **RAM** — This NAS has been upgraded to **32 GB** (the RS1221+ maximum,
  ECC DDR4 SODIMM). That's well above Immich's 8 GB recommended floor, so ML
  jobs (face recognition, smart search, CLIP embedding) have plenty of headroom
  even during the initial bulk import.
- **Filesystem** — Btrfs/ext4 volumes on DSM are fine. Do not put
  `DB_DATA_LOCATION` on a network share or external NTFS drive.
- **`@eaDir`** — DSM sprays these sidecar directories through every shared folder.
  They don't affect this stack (Immich only reads under `UPLOAD_LOCATION`), but if
  you later add External Libraries, you'll need to escape `@eaDir` in Immich's
  exclusion patterns.

## Migrating from Synology Photos

Recommended tool: **[immich-go](https://github.com/simulot/immich-go)** — a
purpose-built CLI that reads Synology Photos exports (including album metadata
from the DSM JSON sidecars) and uploads to Immich in bulk. It preserves
folder-as-album structure and skips duplicates on re-runs.

Rough plan:

1. Bring the stack up empty and create the admin account.
2. Create a non-admin user for yourself and generate an API key.
3. Run `immich-go` from your desktop against a mounted SMB share of `/volume1/photo`,
   or install the binary directly on DSM and run it locally.
4. Verify counts, albums, and a random sample in the Immich web UI.
5. Only *then* stop Synology Photos and reclaim its indexed cache.

**Do not use an External Library for a permanent migration.** External libraries
skip the storage template, block deletes via UI, and mix awkwardly with Synology's
`@eaDir` folders — you'd end up re-importing later anyway.

## Backup

Back up these locations:
- **Library** — `UPLOAD_LOCATION` (the `data/` subdirectory beneath it holds all originals).
- **Database** — **do not** back up the raw Postgres directory. The Immich docs
  explicitly recommend `pg_dump` (see the backup guide below); a raw directory copy
  taken while the DB is running will be inconsistent.

Stop `immich_server` before running the DB dump so the filesystem and DB stay
in sync.

See the official backup guide: <https://docs.immich.app/administration/backup-and-restore>

## Security notes

- Never commit `.env` (it is already in `.gitignore`); only `.env.example` is tracked.
- Use a strong `DB_PASSWORD`.
- Put Immich behind a reverse proxy with TLS for any external access.
