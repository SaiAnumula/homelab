# Backup and restore

This is how I back up the home server and how I check that the backups restore. A backup only counts once a restore from it has worked, so every backup set gets a restore test on a second machine.

Two different roots appear in this document, and confusing them is the fastest way to back up the wrong directory:

- **Live deployment:** on the server, each stack runs from its own Compose project under `/opt/<service>`. Every backup command below runs there. Confirm the actual working directory with `docker compose ls` before trusting any path.
- **Drill checkout:** `~/homelab-restore-drill`, a fresh clone of this repository on the desktop. The restore test runs there, never on the server.

The Compose files in this repository are sanitized templates. Relative binds like `./config` and `./upload/profile` resolve against whichever directory the stack runs from, so the same line means a different absolute path on the server than in the drill.

```bash
export BACKUP_DIR=/path/to/backup/$(date +%F)
mkdir -p "$BACKUP_DIR"
```

`BACKUP_DIR` points at a drive that is not the server's SSD or the 2 TB media HDD.

## What gets backed up

| Data | Location | Method | Why |
|---|---|---|---|
| Immich database | `immich_postgres` container | `pg_dumpall` | Albums, faces, users, and metadata. Losing it means re-importing everything. |
| Immich originals | `/mnt/wd2tb/immich/pics` | `rsync` | The only copy of the photos. |
| Immich profile images | the live profile bind, found by the pre-flight check | `rsync` | Small, and not regenerated. |
| Immich `upload` and `backups` binds | separate live mounts, see the inventory below | **not saved yet** | The pre-flight check stops the backup until these are covered or consciously excluded. |
| Plex database and settings | `/opt/plex/config` | `tar`, with Plex stopped | Watch history, libraries, and server settings. |
| Jellyfin database and settings | `/opt/jellyfin/config` | `tar`, with Jellyfin stopped | Users, watch history, and libraries. |
| `.env` files | each live stack folder under `/opt/<service>` | copied into the backup | Needed to start the stacks again. Store the backup encrypted. |

These are left out on purpose:

- Immich thumbnails and encoded video. Immich regenerates them from the originals.
- Plex and Jellyfin caches and transcode folders.
- The movie and anime library. It can be re-ripped or re-downloaded, and it is too large for the backup drive.

## Pre-flight: mount inventory check

Run this on the server before copying anything. It prints what the live containers actually mount, fails when a required source is missing, and fails when the live Immich stack has a data mount the backup set does not account for.

```bash
set -euo pipefail

# Show every mount the live containers actually use. Read this output
# against the inventory table before continuing.
for c in immich_server immich_postgres; do
  echo "== $c =="
  docker inspect "$c" \
    --format '{{ range .Mounts }}{{ .Source }} -> {{ .Destination }} ({{ .Mode }}){{ "\n" }}{{ end }}'
done

# Every required source from the inventory table must be mounted right now.
mounted=$(docker inspect immich_server \
  --format '{{ range .Mounts }}{{ .Source }}{{ "\n" }}{{ end }}')
for required in /mnt/wd2tb/immich/pics; do
  grep -Fxq "$required" <<< "$mounted" || {
    echo "FAIL: required source is not mounted: $required" >&2
    exit 1
  }
done

# Immich writes originals into anything mounted under /usr/src/app/upload.
# The backup set covers library, thumbs, encoded-video, and profile.
# A separate upload or backups bind is a stop condition: it may hold the
# only copy of user data, and this runbook does not save it.
uncovered=$(docker inspect immich_server \
  --format '{{ range .Mounts }}{{ .Destination }}{{ "\n" }}{{ end }}' \
  | grep '^/usr/src/app/upload/' \
  | grep -vFx -e /usr/src/app/upload/library \
            -e /usr/src/app/upload/thumbs \
            -e /usr/src/app/upload/encoded-video \
            -e /usr/src/app/upload/profile || true)
if [ -n "$uncovered" ]; then
  echo "FAIL: live Immich has upload mounts the backup set does not cover:" >&2
  printf '  %s\n' $uncovered >&2
  echo "Extend the backup set, or record why these hold nothing irreplaceable." >&2
  exit 1
fi
```

When the last check fires, inspect the listed mount. Either add it to "Taking a backup" and to the inventory below, or write down why it holds nothing irreplaceable. A restore drill is not honest unless the pre-flight passed for the exact backup set being tested.

### Mount inventory

Sanitized source-to-destination inventory. "Live" means the running stack under `/opt/<service>` on the server. "Drill" means the clone of this repository on the desktop.

| Data | Live source | Backup destination | Drill restore destination |
|---|---|---|---|
| Immich database | `immich_postgres` container. Its data lives in a Docker volume, not the template's `./postgres` bind, so it is restored from the dump, never by copying files. | `immich-db.sql.gz` | imported with `psql` into the drill's `postgres` service |
| Immich originals (library) | `/mnt/wd2tb/immich/pics` | `immich-library/` | the drill's library volume, repointed at the restored folder |
| Immich profile images | the live bind mounted at `/usr/src/app/upload/profile` | `immich-profile/` | `upload/profile` in the drill checkout |
| Immich `upload` bind | separate live bind under `/usr/src/app/upload` | not covered | not covered |
| Immich `backups` bind | separate live bind under `/usr/src/app/upload` | not covered | not covered |
| Immich thumbnails and encoded video | SSD | not backed up | regenerated from the originals |
| Plex config | `/opt/plex/config` | `plex-config.tar.gz` | `config/` in the drill checkout |
| Jellyfin config | `/opt/jellyfin/config` | `jellyfin-config.tar.gz` | `config/` in the drill checkout |
| `.env` files | `/opt/<service>/.env`, never committed | `<service>.env`, stored encrypted | `.env` in the drill stack folder |

## Taking a backup

Every command in this section runs with `set -euo pipefail`. That is not cosmetic. Without `pipefail`, a pipeline like `pg_dumpall | gzip` hides a failed dump: gzip happily writes a small, checksummable archive of an error message, and the backup looks fine until a restore turns up nothing.

### Immich

Dump the database while Immich is running:

```bash
set -euo pipefail
cd /opt/immich
docker exec -t immich_postgres pg_dumpall --clean --if-exists --username=postgres \
  | gzip > "$BACKUP_DIR/immich-db.sql.gz"
```

Copy the originals and profile images. The profile source is read from the running container, so a moved or renamed bind cannot make this copy the wrong directory:

```bash
set -euo pipefail
rsync -a /mnt/wd2tb/immich/pics/ "$BACKUP_DIR/immich-library/"
profile_src=$(docker inspect immich_server \
  --format '{{ range .Mounts }}{{ if eq .Destination "/usr/src/app/upload/profile" }}{{ .Source }}{{ end }}{{ end }}')
rsync -a "$profile_src"/ "$BACKUP_DIR/immich-profile/"
cp /opt/immich/.env "$BACKUP_DIR/immich.env"
```

Run the database dump before the file copy. A photo uploaded between the two steps then has a file but no database row, which is harmless. The other order leaves a row that points to a missing file.

### Plex

Plex writes to its database constantly, so stop it first:

```bash
set -euo pipefail
cd /opt/plex
docker compose stop plex
tar -czf "$BACKUP_DIR/plex-config.tar.gz" \
  --exclude='Library/Application Support/Plex Media Server/Cache' \
  --exclude='Library/Application Support/Plex Media Server/Crash Reports' \
  config
docker compose start plex
cp .env "$BACKUP_DIR/plex.env"
```

### Jellyfin

```bash
set -euo pipefail
cd /opt/jellyfin
docker compose stop jellyfin
tar -czf "$BACKUP_DIR/jellyfin-config.tar.gz" config
docker compose start jellyfin
cp .env "$BACKUP_DIR/jellyfin.env"
```

### Checksums

```bash
set -euo pipefail
cd "$BACKUP_DIR"
find . -type f ! -name SHA256SUMS -exec sha256sum {} + > SHA256SUMS
```

## Restore test

Run the test on the desktop, not on the server, and in a dedicated clone, not in a checkout that holds anything else. The Compose files set fixed container and project names, and restoring next to the live containers would collide with them or overwrite live data.

The whole drill lives in one directory, so the cleanup at the end can be scoped to it:

```bash
DRILL_DIR=~/homelab-restore-drill
git clone <this repository> "$DRILL_DIR"
```

1. Copy the backup set to the desktop and run `sha256sum -c SHA256SUMS`. Every line must say `OK`.
2. Copy each `<service>.env` from the backup into its stack folder in `$DRILL_DIR` as `.env`. Set `TAILSCALE_IP` to the desktop's address.
3. In `$DRILL_DIR/immich/docker-compose.yml`, point the library volume at the restored `immich-library` folder.
4. Delete the `deploy:` GPU blocks from each Compose file, and drop `-cuda` from the Immich machine-learning image. The desktop has an AMD GPU, and Docker refuses to start a container that reserves an NVIDIA device it cannot find.

Every command below runs with `set -euo pipefail`, so a failed decompression, a failed SQL statement, or a missing file stops the drill instead of producing a half-restored stack that looks finished.

### Immich

```bash
set -euo pipefail
cd ~/homelab-restore-drill/immich
docker compose pull
docker compose create
docker start immich_postgres

# Poll until Postgres accepts connections. A fixed sleep is not a readiness
# signal. (The live stack may name this service `database`; run
# `docker compose config --services` and use the name it prints.)
tries=0
until docker compose exec -T postgres pg_isready --username=postgres --quiet; do
  tries=$((tries + 1))
  if [ "$tries" -ge 60 ]; then
    echo "FAIL: Postgres did not become ready" >&2
    exit 1
  fi
  sleep 1
done

# ON_ERROR_STOP makes psql exit non-zero on the first SQL error, and
# pipefail turns that into a failed drill instead of a silent partial import.
gunzip --stdout "$BACKUP_DIR/immich-db.sql.gz" \
  | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
  | docker compose exec -T postgres psql --dbname=postgres --username=postgres -v ON_ERROR_STOP=1

# The backup saves the profile images, so the restore puts them back
# where Immich reads them.
mkdir -p upload/profile
rsync -a "$BACKUP_DIR/immich-profile/" upload/profile/

docker compose up -d
```

The `sed` line comes from Immich's restore instructions. It fixes the search path that `pg_dumpall` writes, which would otherwise break the vector extensions.

Pass when:

- I can log in with my normal account.
- The photo count under Administration matches the live server.
- Five random photos, including one video, open at full size.
- Albums and named people are present.

### Plex and Jellyfin

```bash
set -euo pipefail
cd ~/homelab-restore-drill/plex && tar -xzf "$BACKUP_DIR/plex-config.tar.gz" && docker compose up -d
cd ~/homelab-restore-drill/jellyfin && tar -xzf "$BACKUP_DIR/jellyfin-config.tar.gz" && docker compose up -d
```

Pass when the libraries, users, and watch history appear. Playback is not part of the test, because the desktop does not have the media drive.

### Clean up

Every command is scoped to the drill directory and the drill's Compose projects. Nothing here touches `/opt/<service>` or any other path, and this section must never run on the server.

```bash
set -euo pipefail
DRILL_DIR=~/homelab-restore-drill
[ -d "$DRILL_DIR/immich" ] || { echo "FAIL: $DRILL_DIR is not the drill checkout" >&2; exit 1; }
for s in immich plex jellyfin; do
  (cd "$DRILL_DIR/$s" && docker compose down --volumes --remove-orphans)
done
rm -rf "$DRILL_DIR/immich/postgres" "$DRILL_DIR/immich/upload" \
       "$DRILL_DIR/plex/config" "$DRILL_DIR/jellyfin/config"
```

## Restore test log

No restore test has been run yet. Each test adds one row to this table.

| Date | Backup set | Immich | Plex | Jellyfin | Time to restore | Notes |
|---|---|---|---|---|---|---|
