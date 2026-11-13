# Hardware

One machine does everything: a Dell laptop, headless, lid closed, shoved somewhere out of the way. It's the only "server" — there's no rack, no NAS, no second node.

## Specs

- **CPU:** Intel Core i5-8250U (4 cores / 8 threads, 1.6 GHz base / 3.4 GHz boost). Kaby Lake Refresh mobile chip.
- **RAM:** 8 GB. This is the real constraint of the whole lab — every design decision below is downstream of it.
- **Storage:**
  - SSD: 116 GB btrfs, subvolumes for `/`, `/home`, `/root`, `/srv`, `/var/*` (CachyOS default layout), compressed with zstd, noatime. Snapshots via snapper.
  - Secondary 29 GB ext4 disk mounted at `/mnt/immich` — Immich's photo library lives here so it can be rebuilt without touching the OS disk.
- **Swap:** 7.6 GiB zram. No swap partition, no swapfile. zram-only because there's no disk headroom to waste and the workload is compressible page cache pressure, not long-idle processes.

## Power situation (the weird part)

The battery is fried — it reports 1% and stays there. The Dell charger also fails its ID check on boot ("AC power adapter wattage and type cannot be determined"), but AC still powers and boosts the CPU fine, so that alert is cosmetic.

What it means in practice: there is zero battery ride-through. A real power blip is an instant hard reset, possibly mid-Postgres-write. Two mitigations:

1. Every container runs with `--restart unless-stopped` / compose restart policies, so boot recovery is automatic.
2. `power-watchdog.sh` (a no-agent cron script) checks `/sys/class/power_supply/AC` and battery status every 15 minutes. It is silent unless power is actually lost, in which case it alerts on WhatsApp — so I find out from my phone before the filesystem can finish booting.

## Why a laptop

Already had it, it draws 10–20 W, has a built-in UPS (well — had), and survives power cuts that would drop a desktop. The fried battery ate that last advantage, but the other two still hold.

## Other box facts

- CachyOS with the stock kernel; ananicy-cpp for auto-nice-ing.
- Boot entries synced with snapper snapshots via `limine-snapper-sync` — a bad update is one boot-menu entry away from a rollback.
- Tailscale is the only remote-access path; see [network.md](network.md).
