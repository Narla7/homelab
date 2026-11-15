# homelab

I have a server at home. It's an old Dell laptop that runs everything I use — photos, music, books, the *arr stack — out of my bedroom, reachable only through Tailscale. This repo documents all of it: hardware, services, compose files, and the reasoning behind the weird parts.

Nothing here is exposed to the internet. No port forwarding, no reverse proxy with a public face. If it's not on Tailscale, it doesn't exist.

## The box

| | |
|---|---|
| Hardware | Dell laptop (headless, lid closed) |
| CPU | Intel Core i5-8250U — 4C/8T @ 1.6–3.4 GHz |
| RAM | 8 GB (7.6 GiB usable) |
| Swap | 7.6 GiB zram — no disk swap at all |
| OS | CachyOS (Arch-based) |
| Storage | 116 GB btrfs (root, snapshots via snapper) + 29 GB ext4 at `/mnt/immich` |
| Power | Battery is fried (pins at 1%) — runs on AC only, with a watchdog that pings me on WhatsApp if power drops |
| Access | Tailscale only — SSH + every web UI |

Details in [docs/hardware.md](docs/hardware.md).

## Services

| Service | What it does | Port |
|---|---|---|
| [Immich](https://immich.app) | Google Photos replacement. Machine learning is fully disabled — this box doesn't have the RAM for it. | 2283 |
| [Navidrome](https://navidrome.org) | Music server, Subsonic API — my phone plays from it with Tempo. | 4533 |
| [Jellyfin](https://jellyfin.org) | Movies (and anything else I point it at), transcoding on CPU | 8096 |
| [Radarr](https://radarr.video) | Movie automation | 7878 |
| [Prowlarr](https://prowlarr.com) | Indexer manager, wires into Radarr + BookOrbit | 9696 |
| [qBittorrent](https://qbittorrent.org) | Download client | 8080 |
| [BookOrbit](https://github.com/bookorbit) | Books + manga, with a read-aloud worker and a request system | 3001 |
| Postgres (pgvector) | BookOrbit's database | internal |
| Postgres (vectorchord) | Immich's database | internal |
| Valkey | Immich's cache/queue | internal |

Per-service notes (why ML is off, how the *arr containers talk to each other, port conflicts) in [docs/services.md](docs/services.md).

## Not in containers (but part of the lab)

- **WhatsApp bridge** — lets me message the server from my phone. Runs on localhost:3000 (which is why BookOrbit got pushed to 3001).
- **Hermes agent** — a local LLM agent that runs on the box, wired to the WhatsApp bridge. It's the one that documented this server and pushes my notes around.
- **power-watchdog.sh** — checks AC/battery status every 15 min, silent unless it actually loses power, then alerts on WhatsApp. Exists because the battery can't ride through a single power blip.

## Repo layout

```
docs/           hardware, services, network, maintenance
compose/        the actual docker-compose files, copied verbatim from the server
```

Secrets never live in this repo — every compose file pulls credentials from a `.env` file (chmod 600) that isn't committed. `compose/*.env.example` shows which variables each stack expects.

## Related

- [Narla7/vault-agent](https://github.com/Narla7/vault-agent) — my Obsidian + local-Ollama agent harness
- [narla.neocities.org](https://narla.neocities.org) — my site, where the vault publishes
