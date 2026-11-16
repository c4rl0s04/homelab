# media

Films and downloads: Jellyfin, Jellyseerr, Radarr, Prowlarr, Bazarr and Byparr, plus qBittorrent behind a ProtonVPN gateway (Gluetun) that forwards a port automatically (qb-port-sync). Dockhand deploys it as a Git stack.

It was migrated on 2026-11-12 from an older single Compose project (`streaming`). The image digests and **container paths did not change**, so application settings such as library paths, the download client and indexer links kept working. Do not start both: their container names and ports overlap.

## Services

| Container | Job | Web UI | Login |
| --- | --- | --- | --- |
| `jellyfin` | Streams the library to TVs, phones and browsers | `jellyfin.casa.home`; port `8096` is open on the LAN for TV and phone apps | Jellyfin's own accounts |
| `jellyseerr` | Where people request films | `jellyseer.casa.home` (one `r`) → `127.0.0.1:5055` | Jellyfin accounts |
| `radarr` | Finds, downloads, renames and files films | `radarr.casa.home` → `127.0.0.1:7878` | Tinyauth (own login `External`) |
| `prowlarr` | Manages the indexers and pushes them to Radarr | `prowlarr.casa.home` → `127.0.0.1:9696` | Tinyauth (own login `External`) |
| `bazarr` | Downloads subtitles for Radarr's films | `bazarr.casa.home` → `127.0.0.1:6767` | Tinyauth (own login off) |
| `byparr` | Solves Cloudflare challenges for Prowlarr (FlareSolverr API), capped at 2 CPUs | none; `127.0.0.1:8191` | none |
| `gluetun` | ProtonVPN WireGuard tunnel and firewall; owns qBittorrent's network | qBittorrent's UI is published here: `qbit.casa.home` → `gluetun:8080`, `127.0.0.1:8080` | none of its own |
| `qbittorrent` | Downloads torrents, only through the VPN | through `gluetun:8080` | Tinyauth; its own login is skipped for the `proxy` subnet (`172.21.0.0/16` whitelist) |
| `qb-port-sync` | Copies ProtonVPN's forwarded port into qBittorrent's listening port every 5 minutes | none | uses `QBIT_API_KEY` |

## How the apps talk to each other

Every container is on infra's `proxy` network (this stack's default network *is* `proxy`), so apps reach each other by container name. These links are stored in each app's own database, not in Git; check them after a restore or a container rename.

| From | To | Address set in the app | Where |
| --- | --- | --- | --- |
| Jellyseerr | Jellyfin | `jellyfin:8096` | Jellyseerr settings → Jellyfin |
| Jellyseerr | Radarr | `radarr:7878`, root folder `/data/media/movies` | Jellyseerr settings → Services |
| Prowlarr | Radarr | Prowlarr `http://prowlarr:9696`, Radarr `http://radarr:7878`; Prowlarr pushes its indexers to Radarr as `http://prowlarr:9696/<id>/` | Prowlarr → Settings → Apps |
| Prowlarr | Byparr | `http://byparr:8191`, type FlareSolverr | Prowlarr → Settings → Indexers (proxies) |
| Radarr | qBittorrent | host `gluetun`, port `8080`, category `movies` | Radarr → Settings → Download Clients |
| Bazarr | Radarr | `radarr:7878` | Bazarr → Settings → Radarr |

### Paths

Radarr and qBittorrent see the download folder under different names, so Radarr has a **remote path mapping**: host `gluetun`, remote `/downloads/` → local `/data/downloads/`.

| Host folder | qBittorrent | Radarr / Bazarr | Jellyfin |
| --- | --- | --- | --- |
| `${MEDIA_ROOT}/downloads` | `/downloads` (unfinished in `incomplete/`, finished in `movies/`) | `/data/downloads` (also mounted as `/downloads`) | — |
| `${MEDIA_ROOT}/media/movies` | — | `/data/media/movies` (root folder; also mounted as `/movies`, an older root folder) | `/media/movies` (library) |

`/data` is all of `MEDIA_ROOT` in one mount, so downloads and the library are on the same filesystem as Radarr sees them.

### VPN and the forwarded port

- `qbittorrent` and `qb-port-sync` use `network_mode: service:gluetun`: they have no network of their own. Gluetun's firewall lets traffic out only through the WireGuard tunnel, so if the VPN drops, downloads stop instead of leaking. `FIREWALL_INPUT_PORTS=8080` lets npm reach the web UI.
- ProtonVPN forwards one port for incoming peers and may change it on every reconnect. Gluetun writes it to `/tmp/gluetun/forwarded_port` (the shared volume `gluetun_port`). qb-port-sync reads it and sets qBittorrent's listening port through the API.
- Gluetun's health check allows 2 minutes to connect (`start_period`). qBittorrent waits until it is healthy, which avoids a failed first start.
- Recreating `gluetun` cuts the network of the two containers inside it. Restart them after it; a stack redeploy does this in order.

## Deploy with Dockhand

Git stack `media`: repository `homelab`, branch `main`, compose file `stacks/media/compose.yaml`. Stack variables, also listed in `.env.example`:

| Variable | Windows PC value |
| --- | --- |
| `DATA_ROOT` | `E:/homelab/data` |
| `MEDIA_ROOT` | `E:/homelab/media` (holds `downloads/` and `media/`) |
| `TZ` | `Europe/Madrid` |
| `PUID` / `PGID` | `1000` |
| `PROTON_WG_PRIVATE_KEY` | Secret. A ProtonVPN WireGuard configuration with port forwarding (NAT-PMP) on: its `PrivateKey` |
| `QBIT_API_KEY` | Secret. qBittorrent's Web UI API key (`WebUI\APIKey` in its config) |

Requires `infra` first: it provides the names, the login and the `proxy` network this stack joins.

## Data

| Path | Content | Backed up |
| --- | --- | --- |
| `${DATA_ROOT}/media/<app>/` | Settings and databases of each app (`radarr/radarr.db`, `prowlarr/prowlarr.db`, `jellyseerr/settings.json`, `qbittorrent/qBittorrent/qBittorrent.conf`, …) | Yes (Backrest; log folders excluded) |
| `${DATA_ROOT}/media/jellyfin-cache/` | Jellyfin image and transcode cache; regenerates | Yes (small) |
| `${MEDIA_ROOT}/media/` | Films | No: re-downloadable; protect separately if needed |
| `${MEDIA_ROOT}/downloads/` | qBittorrent downloads | No |

## Checklist

- [x] Cutover done and verified 2026-11-12:
  - backup taken first, and app configs copied with identical file counts and sizes;
  - Gluetun healthy on its own ProtonVPN exit IP, and the forwarded port synced into qBittorrent;
  - all web UIs and names respond;
  - Jellyfin keeps its server identity and its library (`/media/movies`).
- [x] Own logins off behind Tinyauth (2026-11-13 and 11-15; details in `stacks/infra/README.md`).
- [x] Byparr capped at 2 CPUs (2026-11-16): its headless Chrome spiked past that and slowed other stacks.
- Known issue: Radarr and Prowlarr log SSL errors reaching their update and notification servers. The old logs show the same before the migration, so it is a host network issue, not the cutover.
