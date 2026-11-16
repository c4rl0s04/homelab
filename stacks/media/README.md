# media

Films and downloads: Jellyfin, Jellyseerr, Radarr, Prowlarr, Bazarr, Byparr, and qBittorrent behind a ProtonVPN gateway (gluetun) with automatic port forwarding (qb-port-sync). Deployed by Dockhand as a Git stack.

Migrated on 2026-11-12 from an older single Compose project. Image digests and **container paths are unchanged**, so application settings (library paths, download client `gluetun:8080`, indexer links) keep working.

## Requires

- `infra` deployed first: this stack joins its `proxy` network, where Nginx Proxy Manager forwards `jellyfin`, `jellyseer`, `radarr`, `prowlarr`, `bazarr` and `qbit` (to `gluetun:8080`) under `*.casa.home`. Only Jellyfin publishes its port to the LAN (TV and phone apps); the other web UIs listen on `127.0.0.1` and are reached through npm and Tinyauth.

## Deploy with Dockhand

Git stack `media`, compose file `stacks/media/compose.yaml`, variables from `.env.example`. `PROTON_WG_PRIVATE_KEY` and `QBIT_API_KEY` are secrets.

## Data

| Path | Content | Backed up |
| --- | --- | --- |
| `${DATA_ROOT}/media/<app>/` | Configuration and databases of each app | Yes (Backrest) |
| `${DATA_ROOT}/media/jellyfin-cache/` | Jellyfin image/transcode cache, regenerates | Yes (small) |
| `${MEDIA_ROOT}/media/` | Films | No: re-downloadable; protect separately if needed |
| `${MEDIA_ROOT}/downloads/` | qBittorrent downloads | No |

## Checklist

- [x] Cutover done and verified 2026-11-12: backup taken first; app configs copied with identical file counts and sizes; gluetun healthy on its own ProtonVPN exit IP, forwarded port synced into qBittorrent; all web UIs and `*.home` names respond; Jellyfin keeps its server identity and its library (`/media/movies`). Radarr and Prowlarr log SSL errors reaching their update/notification servers; the old logs show the same errors before the migration (host-side network issue, not caused by the cutover).
