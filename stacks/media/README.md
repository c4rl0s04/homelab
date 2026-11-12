# media

Films and downloads: Jellyfin, Jellyseerr, Radarr, Prowlarr, Bazarr, Byparr, and qBittorrent behind a ProtonVPN gateway (gluetun) with automatic port forwarding (qb-port-sync). Deployed by Dockhand as a Git stack.

Migrated from the old `streaming` stack (`<old-project>`, repo `previous-project`). Image digests and **container paths are unchanged**, so application settings (library paths, download client `gluetun:8080`, indexer links) keep working.

## Requires

- `infra` deployed first: this stack joins its `proxy` network, where Nginx Proxy Manager forwards `jellyfin.home`, `radarr.home`, `qbit.home` (to `gluetun:8080`) and `jellyseer.home`.

## Deploy with Dockhand

Git stack `media`, compose file `stacks/media/compose.yaml`, variables from `.env.example`. `PROTON_WG_PRIVATE_KEY` and `QBIT_API_KEY` are secrets.

## Data

| Path | Content | Backed up |
| --- | --- | --- |
| `${DATA_ROOT}/media/<app>/` | Configuration and databases of each app | Yes (Backrest) |
| `${DATA_ROOT}/media/jellyfin-cache/` | Jellyfin image/transcode cache, regenerates | Yes (small) |
| `${MEDIA_ROOT}/media/` | Films | No: re-downloadable; protect separately if needed |
| `${MEDIA_ROOT}/downloads/` | qBittorrent downloads | No |

## Cutover from the old stack

1. Backup run in Backrest.
2. Old stack: `docker compose stop` then `docker compose rm -f` in `<old-project>` (its config and data stay untouched as rollback).
3. Copy each app folder from `<old-project>\configs\` to `${DATA_ROOT}\media\` (`qbittorrent`, `radarr`, `prowlarr`, `jellyfin`, `jellyseerr`, `bazarr`).
4. Redeploy `infra` (Nginx drops the old `streaming_default` network), then deploy `media` from Dockhand.
5. Verify: gluetun healthy with a ProtonVPN IP and a forwarded port synced into qBittorrent; Radarr and Prowlarr connected; Jellyfin libraries and users intact; all `*.home` names respond.

**Rollback:** remove the `media` stack in Dockhand (keep files), then `docker compose up -d` the media services in `<old-project>` (and re-add `streaming_default` to npm if needed).

## Checklist

- [ ] Cutover done and verified.
- [ ] Old stack folder retired after a few days of normal use.
