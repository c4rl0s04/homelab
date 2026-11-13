# portal

[Homepage](https://github.com/gethomepage/homepage): one page with every app, its up/down status and live widgets. Open it at `http://portal.casa.home` (behind Tinyauth: the shared login is asked here once).

## Configuration lives in `compose.yaml`

Homepage's YAML files (`settings`, `services`, `widgets`, `bookmarks`) are inline Compose `configs`. Compose copies them into the container, so:

- the whole dashboard is versioned in Git and moves with the repository;
- no host path is needed. Dockhand runs Compose from its own checkout (`/app/data/stacks/...`), so a relative bind mount such as `./config` would point at a path that does not exist on the host.

Change the dashboard by editing `compose.yaml` and push. Then in Dockhand press **Desmontar** (removes only the container; Homepage keeps no data) and **Desplegar**: a plain sync pulls the new file but Compose does not recreate the container when only inline `configs` content changes (seen 2026-11-13 with Dockhand 1.0.57). Changes to stack variables (widget keys) apply with a normal redeploy. Inside the inline content, write `$$` for a literal `$` (Compose interpolates `${...}`).

No Docker socket: status dots come from `siteMonitor` checks against each container on the `proxy` network.

## Deploy with Dockhand

Git stack `portal`: repository `homelab`, branch `main`, compose file `stacks/portal/compose.yaml`.

| Variable | Value |
| --- | --- |
| `TZ` | `Europe/Madrid` |
| `JELLYFIN_API_KEY` | Secret. Jellyfin → Panel de control → Claves API → `+` |
| `JELLYSEERR_API_KEY` | Secret. Jellyseerr → Settings → General → API Key |
| `RADARR_API_KEY` | Secret. Radarr → Settings → General → API Key |
| `PROWLARR_API_KEY` | Secret. Prowlarr → Settings → General → API Key |
| `BAZARR_API_KEY` | Secret. Bazarr → Settings → General → API Key |

The keys are optional: without one, that widget shows an error and its link and status still work. Optional: `PORTAL_PORT` (3004).

npm proxy host: `portal.casa.home` → `homepage:3000`, behind Tinyauth. `HOMEPAGE_ALLOWED_HOSTS` must list every name used to open it.

## Data and backup

None: the configuration is in Git and the keys are in Dockhand. Homepage only writes logs inside the container.

## Checklist

- [x] Trial run outside Dockhand (2026-11-12, `v2.7.0`): page renders, all groups and status checks load; widgets without keys show "API error" as expected.
- [x] Deployed as a Git stack (2026-11-12); `portal.home` answers; Uptime Kuma checks it with header `Host: portal.home` (Homepage rejects other host names).
- [x] Widget keys (2026-11-13): Jellyfin (widget `version: 2` for Jellyfin 12), Jellyseerr, Radarr, Prowlarr and Bazarr show data.
