# Network

## Access model: Tailscale-only

- The server joins a tailnet. SSH and every web UI are reachable only from my other devices on the same tailnet.
- **Zero public exposure:** no port forwarding on the router, no reverse proxy, no DDNS. The web UIs technically bind `0.0.0.0` but nothing routes to them from outside the LAN/tailnet.
- SSH is the raw OpenSSH server on 22, reachable from LAN and Tailscale.

## The MagicDNS quirk

This server's MagicDNS is broken — `tailscale status` shows a health warning because systemd-resolved and NetworkManager are wired together incorrectly. Net effect:

- `http://server:<port>` and `*.ts.net` hostnames **do not resolve** even though the raw tailnet IP works fine.
- Phone clients configured with a hostname keep showing "app down" after every reboot.

So all clients (Tempo, Immich app) are configured with the **raw tailnet IP + port**, and that's the rule for anything new too.

## Ports actually listening

```
22     ssh          LAN + tailscale
2283   immich       tailscale
4533   navidrome    tailscale
7878   radarr       tailscale
8080   qbittorrent  tailscale
9696   prowlarr     tailscale
3001   bookorbit    tailscale
3000   whatsapp bridge  localhost only
6881   qbittorrent bittorrent tcp+udp
```

## Inter-container networking

Stacks talk over Docker networks, addressed by container name, not IPs:

- BookOrbit joins `prowlarr_default` (declared external in its compose) → Prowlarr URL inside BookOrbit is `http://prowlarr:9696`.
- Radarr → qBittorrent wired the same way; the "Host" field in the *arr UIs is the container name.
- Immich's server env uses compose service names (`database`, `redis`) for DB_HOSTNAME/REDIS_HOSTNAME.

## What I'd tell a stranger

Don't publish a *arr WebUI or Immich to the internet. Tailscale (or WireGuard, or any VPN) plus raw IPs is enough for one user, has no attack surface, and requires no reverse-proxy ceremony.
