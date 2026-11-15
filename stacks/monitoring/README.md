# monitoring

[Uptime Kuma](https://github.com/louislam/uptime-kuma): checks every web service and alerts when one stops answering.

Its own stack on purpose: it must keep running while the stacks it watches (including `infra`) are stopped or redeployed.

## Deploy with Dockhand

Git stack `monitoring`: repository `homelab`, branch `main`, compose file `stacks/monitoring/compose.yaml`. Stack variables (no secrets):

| Variable | Windows PC value |
| --- | --- |
| `DATA_ROOT` | `E:/homelab/data` |
| `TZ` | `Europe/Madrid` |

Optional: `UPTIME_KUMA_PORT` (3003). After deploying, open `http://status.casa.home` (npm proxy host → `uptime-kuma:3001`, websockets on, behind Tinyauth; its own login is disabled).

## Monitors

Check services by container name on the `proxy` network, not by `*.casa.home` (that would test DNS and npm instead of the app):

| Monitor | URL |
| --- | --- |
| Jellyfin | `http://jellyfin:8096/health` |
| Jellyseerr | `http://jellyseerr:5055` |
| Radarr | `http://radarr:7878/ping` |
| qBittorrent | `http://gluetun:8080` |
| TREK | `http://trek:3000/api/health` |
| Dockhand | `http://dockhand:3000` |
| Backrest | `http://backrest:9898` |
| AdGuard | `http://adguardhome:3000` |
| npm | `http://npm:81` |
| Prowlarr | `http://prowlarr:9696/ping` |
| Bazarr | `http://bazarr:6767` |
| PairDrop | `http://pairdrop:3000` |
| Stirling-PDF | `http://stirling-pdf:8080` |

One DNS monitor resolves `jellyfin.casa.home` against the host IP (`192.168.1.10`) to cover AdGuard itself. It goes through the Windows host network, so it fails while ProtonVPN is connected on the host (which blocks DNS to other servers); LAN devices are not affected. Expect it to recover on the mini PC.

## Status page

Status page `homelab` (slug) lists every monitor; the portal's Uptime Kuma widget reads it (`/status/homelab`). Add new monitors to it, or the widget will not count them.

## Data and backup

`${DATA_ROOT}/uptime-kuma` (SQLite: monitors, history, account). Backed up with the rest of `DATA_ROOT`.

## Checklist

- [x] Deployed as a Git stack (2026-11-12); `status.home` answers through npm. Database: SQLite (chosen at first start).
- [x] Monitors for every service above, plus Prowlarr, Bazarr and the DNS check, are green (12/12, 2026-11-12). Two retries before a service counts as down.
- [x] Change → push → sync test (2026-11-12): `stop_grace_period: 30s` merged in PR #9, synced in Dockhand; the container was recreated with `StopTimeout=30` and all 12 monitors and their history survived.
