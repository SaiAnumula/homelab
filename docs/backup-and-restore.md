# Backup and restore

This is how I back up the home server and how I check that the backups restore. A backup only counts once a restore from it has worked, so every backup set gets a restore test on a second machine.

Commands assume the stacks live in `~/homelab/<service>` on the server and that `BACKUP_DIR` points at a drive that is not the server's SSD or the 2 TB media HDD.

```bash
export BACKUP_DIR=/path/to/backup/$(date +%F)
mkdir -p "$BACKUP_DIR"
```

## What gets backed up

| Data | Location | Method | Why |
|---|---|---|---|
| Immich database | `immich_postgres` container | `pg_dumpall` | Albums, faces, users, and metadata. Losing it means re-importing everything. |
| Immich originals | `/mnt/wd2tb/immich/pics` | `rsync` | The only copy of the photos. |
| Immich profile images | `~/homelab/immich/upload/profile` | `rsync` | Small, and not regenerated. |
| Plex database and settings | `~/homelab/plex/config` | `tar`, with Plex stopped | Watch history, libraries, and server settings. |
| Jellyfin database and settings | `~/homelab/jellyfin/config` | `tar`, with Jellyfin stopped | Users, watch history, and libraries. |
| `.env` files | Each stack folder | Copied into the backup | Needed to start the stacks again. Store the backup encrypted. |

These are left out on purpose:

- Immich thumbnails and encoded video. Immich regenerates them from the originals.
- Plex and Jellyfin caches and transcode folders.
- The movie and anime library. It can be re-ripped or re-downloaded, and it is too large for the backup drive.

## Taking a backup

### Immich

Dump the database while Immich is running:

```bash
cd ~/homelab/immich
docker exec -t immich_postgres pg_dumpall --clean --if-exists --username=postgres \
  | gzip > "$BACKUP_DIR/immich-db.sql.gz"
```

Copy the originals and profile images:

```bash
rsync -a /mnt/wd2tb/immich/pics/ "$BACKUP_DIR/immich-library/"
rsync -a ~/homelab/immich/upload/profile/ "$BACKUP_DIR/immich-profile/"
cp ~/homelab/immich/.env "$BACKUP_DIR/immich.env"
```

Run the database dump before the file copy. A photo uploaded between the two steps then has a file but no database row, which is harmless. The other order leaves a row that points to a missing file.

### Plex

Plex writes to its database constantly, so stop it first:

```bash
cd ~/homelab/plex
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
cd ~/homelab/jellyfin
docker compose stop jellyfin
tar -czf "$BACKUP_DIR/jellyfin-config.tar.gz" config
docker compose start jellyfin
cp .env "$BACKUP_DIR/jellyfin.env"
```

### Checksums

```bash
cd "$BACKUP_DIR" && find . -type f ! -name SHA256SUMS -exec sha256sum {} + > SHA256SUMS
```

## Restore test

Run the test on the desktop, not the server. The Compose files set fixed container names, and restoring next to the live containers would collide with them or overwrite live data.

1. Copy the backup set to the desktop and run `sha256sum -c SHA256SUMS`. Every line must say `OK`.
2. Clone this repository and copy each `<service>.env` from the backup into its stack folder as `.env`. Set `TAILSCALE_IP` to the desktop's address.
3. In `immich/docker-compose.yml`, point the library volume at the restored `immich-library` folder.
4. Delete the `deploy:` GPU blocks from each Compose file, and drop `-cuda` from the Immich machine-learning image. The desktop has an AMD GPU, and Docker refuses to start a container that reserves an NVIDIA device it cannot find.

### Immich

```bash
cd ~/homelab/immich
docker compose pull
docker compose create
docker start immich_postgres
sleep 10
gunzip --stdout "$BACKUP_DIR/immich-db.sql.gz" \
  | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
  | docker exec -i immich_postgres psql --dbname=postgres --username=postgres
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
cd ~/homelab/plex && tar -xzf "$BACKUP_DIR/plex-config.tar.gz" && docker compose up -d
cd ~/homelab/jellyfin && tar -xzf "$BACKUP_DIR/jellyfin-config.tar.gz" && docker compose up -d
```

Pass when the libraries, users, and watch history appear. Playback is not part of the test, because the desktop does not have the media drive.

### Clean up

```bash
for s in immich plex jellyfin; do (cd ~/homelab/$s && docker compose down -v); done
rm -rf ~/homelab/immich/postgres ~/homelab/plex/config ~/homelab/jellyfin/config
```

## Restore test log

No restore test has been run yet. Each test adds one row to this table.

| Date | Backup set | Immich | Plex | Jellyfin | Time to restore | Notes |
|---|---|---|---|---|---|---|
