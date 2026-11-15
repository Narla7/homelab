# Services

Every service is a Docker container in its own `~/<service>/` directory with its own `docker-compose.yml`. Compose plugin, one stack per app, host bind-mounts for anything that must survive container recreation.

## Port map

| Port | Service |
|---|---|
| 2283 | Immich |
| 4533 | Navidrome |
| 8096 | Jellyfin |
| 7878 | Radarr |
| 9696 | Prowlarr |
| 8080 | qBittorrent WebUI (6881 = BitTorrent) |
| 3001 | BookOrbit |
| 3000 | **not free** — WhatsApp bridge (localhost only) |

All WebUIs publish on `0.0.0.0` but the only routable path to them is Tailscale — no router port-forwards exist.

## Immich

- Server + valkey + postgres (pinned `14-vectorchord` image) under compose, photos on `/mnt/immich`.
- **Machine learning is entirely disabled** via `IMMICH_MACHINE_LEARNING_ENABLED: 'false'` — 8 GB RAM can't carry smart search / face recognition / OCR, and half of Immich's value is the plain backup and timeline anyway. This is the deliberate trade.
- Update policy: **never blind-pull.** Immich has breaking changes between releases and its postgres sidecar is pinned per release. Copy tags out of the new release's compose file first. Other stacks are safe to pull-blind.
- The sidecar images are never bumped for fun — major Postgres upgrades are manual pg_upgrade work.

## Navidrome

Music server over the Subsonic API. Library lives in `~/Music`. Clients (Tempo on my phone) hit it directly on the Tailscale IP. Flac-to-mp3 conversions handled by a `flac2mp3.sh` helper in `$HOME`.

## Jellyfin

Movie/media server on **8096**, official `jellyfin/jellyfin` image run with `user: 1000:1000` (the official image manages its own uid, no PUID/PGID envs). Config and cache live in `~/jellyfin/{config,cache}`, both user-owned so the database survives container recreation.

Media is bind-mounted **read-only**: `~/media` (`/media` — where Radarr is meant to import) and `~/torrents` (`/torrents`). Since the mounts are `:ro`, Jellyfin writes all metadata/NFO/trickplay to `/config` instead of next to the files, which is the behavior I want anyway. No hardware acceleration — transcoding is CPU-only ffmpeg, fine on the i5 for one stream.

No DLNA/host networking — published port only, everything reaches it over Tailscale with the raw IP.

## The *arr stack + qBittorrent

- **Prowlarr** (9696) manages indexers and hands them to **Radarr** (7878, movies) and **BookOrbit**.
- **qBittorrent** (8080) is the only download client — BitTorrent only, no usenet. Downloads land in `~/torrents`.
- Cross-stack wiring uses Docker networks, not IPs: the dependent container joins the other stack's network and the "Host" field in the *arr UI is the **container name** (e.g. `http://prowlarr:9696`). `localhost` inside a container points at that container — the classic misconfig.
- The network connections are re-declared in compose as external networks, because `docker network connect` is runtime-only and dies on the next `compose up`.
- LinuxServer images run with `PUID/PGID=1000` env (never `--user`), `TZ` set, `--restart unless-stopped`.

## BookOrbit

Books + manga reader on host port **3001** — port 3000 is taken by the WhatsApp bridge. Postgres with the pgvector image as its database, plus a read-aloud worker (Kokoro TTS, CPU-only) on a private compose network. Its indexer/download-client credentials refuse to save without `BOOK_REQUEST_ENCRYPTION_KEY` set, and the app service joins `prowlarr_default` so Prowlarr settings use `http://prowlarr:9696` with Prowlarr's API key.

Manga lives in `~/manga`, books in `~/Books` / `~/Calibre Library`, all bind-mounted read-only into the container.

## Not containerized

- **WhatsApp bridge** — Node process on `localhost:3000`, self-chat mode, backed by a Hermes agent running as a user service (`hermes-gateway.service`). This is how the server talks to my phone: power-loss alerts, agent chat, anything I want pushed to me.
- **Hermes agent** — local LLM agent on the box itself. It maintains the vault, runs maintenance checks, and documented this very repo.

## Secrets handling

Every stack pulls credentials from a `.env` next to its compose file (chmod 600), generated from an even-more-private source file for Immich. Nothing is baked into compose files, nothing is committed here. See `compose/*.env.example` for the variable names each stack expects.
