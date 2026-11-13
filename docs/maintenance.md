# Maintenance

## Updating a stack

Per stack:

```sh
cd ~/<service>
docker compose pull && docker compose up -d && docker image prune -f
```

`pull` is the update check ("up to date" = done). `up -d` recreates only containers whose image changed — databases stay up. Rollback for `latest`-tagged apps: the old image survives the pull until `prune`, so set the old image ID as the image var in `.env` and `up -d` again.

**Immich exception:** never blind-pull. Breaking changes between releases, and the postgres/vectorchord sidecar is pinned per release — copy the tags out of the new release's compose file first.

Sidecar rule: postgres/redis/valkey images are never bumped casually. Major Postgres versions mean manual pg_upgrade.

## Verifying after any change

`docker ps` alone is a lie — a crash loop reads as "Up" if you squint. The real checklist:

1. `docker ps` — status says `Up`, **not** `Restarting (N)`.
2. `curl -s -o /dev/null -w '%{http_code}' http://localhost:<port>` — any HTTP answer, even a 302 to a login page, means the app is alive.
3. `docker logs <name> --since 10m` — no fatal lines, and for scan-type apps, actual work being logged.

## Reboots & snapshots

- Root filesystem is btrfs with snapper snapshots; boot entries are auto-synced to the snapshot list (`limine-snapper-sync`), so a bad update is rolled back from the boot menu.
- zram-only swap: nothing to fsck, nothing to wait on, but RAM pressure is absorbed by compression — watch `free -h`'s swap column, not disk.
- After a reboot, Immich's Postgres logs `connect ECONNREFUSED` for a minute or two while it initializes — that's the boot race, not a bug. Same for anything that starts before the database.

## Disks

| Device | Mount | FS | Role |
|---|---|---|---|
| SSD | `/` (+ /home, /var/* subvolumes) | btrfs (zstd, noatime) | OS + app data |
| secondary 29 GB | `/mnt/immich` | ext4 | Immich photo library (`nofail` in fstab) |

The Immich data disk is separate on purpose: the library can be moved/rebuilt without touching the OS.

## Known scars

- Fried battery → hard-reset on any power blip; watchdog covers alerting, restart policies cover recovery.
- MagicDNS broken on this node → always configure clients with the raw tailnet IP.
- Port 3000 is spoken for (WhatsApp bridge) → new web apps start at 3001+.
