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
- `IMMICH_VERSION` — `release` for latest stable, or pin a version like `v2.1.0`

### 2. Create the data directories

```bash
mkdir -p /volume1/docker/immich/library
mkdir -p /volume1/docker/immich/postgres
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

## Hardware acceleration (optional)

This setup runs CPU-only. To enable GPU acceleration (e.g. NVIDIA, matching the
Jellyfin setup) you need to add the upstream hardware-acceleration compose files
and uncomment the `extends:` blocks in `compose.yaml`:

- Transcoding: <https://docs.immich.app/features/hardware-transcoding>
- Machine learning: <https://docs.immich.app/features/ml-hardware-acceleration>

Get the matching files from the
[current release](https://github.com/immich-app/immich/releases/latest):
`hwaccel.transcoding.yml` and `hwaccel.ml.yml`.

## Backup

Back up these locations:
- **Database** — `DB_DATA_LOCATION` (or use Immich's recommended `pg_dumpall` procedure)
- **Library** — `UPLOAD_LOCATION`

See the official backup guide: <https://docs.immich.app/administration/backup-and-restore>

## Security notes

- Never commit `.env` (it is already in `.gitignore`); only `.env.example` is tracked.
- Use a strong `DB_PASSWORD`.
- Put Immich behind a reverse proxy with TLS for any external access.
