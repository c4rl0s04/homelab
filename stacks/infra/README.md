# infra

Shared infrastructure used by every stack. **Managed by Dockhand as a Git stack** (the manager itself lives in `stacks/dockhand`). Stopping or redeploying this stack interrupts home DNS for a few seconds; the router's fallback resolver keeps Internet working meanwhile.

| Service | Purpose | Admin UI | Status |
| --- | --- | --- | --- |
| AdGuard Home | Home DNS: ad blocking and `*.casa.home` names (DNS rewrites) for every device | `127.0.0.1:${ADGUARD_UI_PORT:-3000}` | Running |
| Nginx Proxy Manager | Reverse proxy on 80/443: forwards `*.casa.home` to each stack's web service | `:${NPM_ADMIN_PORT:-81}` | Running |
| Tinyauth | One shared login for the web UIs (npm asks it before serving a protected name) | `auth.casa.home` | Running |
| Backrest | Encrypted, deduplicated backups (restic) of `DATA_ROOT`; bulk media is separate | `127.0.0.1:${BACKREST_PORT:-9898}` | Running |

Tailscale is planned; portal and the other applications are separate stacks (see the root README). The other stacks need this one running: without AdGuard and npm no `*.casa.home` name answers, and without Tinyauth every protected name stops answering.

## Deploy with Dockhand

Git stack `infra`: repository `homelab`, branch `main`, compose file `stacks/infra/compose.yaml`. Stack variables (no secrets in this stack):

| Variable | Windows PC value |
| --- | --- |
| `DATA_ROOT` | `E:/homelab/data` |
| `BACKUP_ROOT` | `C:/homelab-backups` |
| `TZ` | `Europe/Madrid` |
| `DNS_BIND_IP` | The host's LAN IP, reserved in the router (e.g. `192.168.1.10`) |

Optional: `ADGUARD_UI_PORT` (3000), `NPM_ADMIN_PORT` (81), `BACKREST_PORT` (9898).

**Without Dockhand** (recovery, or a new host before Dockhand is up): `docker compose --env-file ../../hosts/<host>.env up -d` from this folder. Dockhand then shows the stack as untracked until it is registered again.

## How other stacks use it

- **DNS** works at network level: the router hands out this host as DNS server, and AdGuard rewrites `<app>.casa.home` to the host IP. Keep a fallback resolver in the router (e.g. `1.1.1.1`) so the home network survives if this host is down.
- **Upstream DNS uses an IP literal** (`https://9.9.9.10/dns-query`, Quad9 unfiltered). On the Windows PC, ProtonVPN blocks plain DNS to anything but its own resolver, so AdGuard could not bootstrap `dns10.quad9.net` by name. The Windows host itself also cannot query AdGuard while ProtonVPN is connected; other devices can. Revisit on the mini PC (no desktop VPN there).
- **Router DHCP** hands out the host IP first and `1.1.1.1` as a fallback. Devices may pick `1.1.1.1`, which does not know these names (confirmed 2026-11-12: a phone works with DNS set manually to the host IP). Kept on purpose while AdGuard runs on a PC that is not always on; remove the fallback once the mini PC hosts DNS. The host's IP is reserved in the router (2026-11-13), so the DNS address and `DNS_BIND_IP` stay valid.
- **Port 53 on Windows:** the "Internet Connection Sharing" service (`SharedAccess`) holds UDP `0.0.0.0:53` after every reboot; Docker/WSL networking relies on it, so do not disable it. AdGuard binds the LAN address instead (`DNS_BIND_IP`, e.g. `192.168.1.10:53`), which takes precedence over the wildcard (verified 2026-11-13). The address must stay fixed: reserve it in the router. On the mini PC (Linux) leave `DNS_BIND_IP` unset.
- **Names:** the AdGuard rewrite `*.home → <host IP>` covers every `<app>.casa.home`; npm maps each name to a container. Names use two labels before `home` because the shared login cookie needs a parent domain (`casa.home`); with plain `*.home` Tinyauth limits the cookie to `auth.home` (tested 2026-11-13). The old `*.home` proxy hosts were removed.

  | Name | Target | Login |
  | --- | --- | --- |
  | `portal.casa.home` | `homepage:3000` | Tinyauth |
  | `radarr.casa.home` | `radarr:7878` | Tinyauth |
  | `prowlarr.casa.home` | `prowlarr:9696` | Tinyauth |
  | `bazarr.casa.home` | `bazarr:6767` | Tinyauth |
  | `qbit.casa.home` | `gluetun:8080` | Tinyauth |
  | `status.casa.home` | `uptime-kuma:3001` | Tinyauth |
  | `backrest.casa.home` | `backrest:9898` | Tinyauth |
  | `adguard.casa.home` | `adguardhome:3000` | Tinyauth |
  | `dockhand.casa.home` | `dockhand:3000` | Tinyauth, then Dockhand's own (root-equivalent: two locks) |
  | `pdf.casa.home` | `stirling-pdf:8080` | Tinyauth |
  | `beszel.casa.home` | `beszel:8090` | Tinyauth; Beszel trusts its `Remote-Email` header (see `stacks/monitoring/README.md`) |
  | `browser.casa.home` | `neko:8080` | Tinyauth (Neko's own login is off: every visitor is admin) |
  | `drop.casa.home` | `pairdrop:3000` | None: open to every device at home, guests included |
  | `jellyfin.casa.home` | `jellyfin:8096` | Jellyfin's own (TV and phone apps cannot use Tinyauth) |
  | `jellyseer.casa.home` | `jellyseerr:5055` | Jellyfin account |
  | `trek.casa.home` | `trek:3000` | TREK's own accounts (shared with other people) |
  | `npm.casa.home` | `npm:81` | npm's own: it configures Tinyauth, so it must not depend on it |
  | `auth.casa.home` | `tinyauth:3000` | The login page itself |

- **Shared login (Tinyauth):** a protected proxy host has this in npm → Edit → Advanced. Before serving a page, npm asks Tinyauth (`auth_request`); Tinyauth answers 200 when the browser's session cookie is valid, otherwise 401 with the login address, which npm turns into a redirect. Users live in `${DATA_ROOT}/tinyauth/users.txt` (`docker run --rm -it ghcr.io/tinyauthapp/tinyauth:v5.2.0 user create --interactive`, paste the line, redeploy). Sessions last 7 days (`TINYAUTH_SESSION_SECONDS`). Apps behind it have their own login turned off; their ports are published on `127.0.0.1` only so the LAN cannot bypass it. If Tinyauth is down, protected names stop answering; from the host each app still opens at `127.0.0.1:<port>`.

  ```nginx
  location / {
    include conf.d/include/proxy.conf;
    # Defining proxy_set_header here stops npm's server-level websocket headers from being inherited,
    # so they are repeated: without them Neko never loads and Kuma or Dockhand fall back to polling.
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
    auth_request /tinyauth;
    auth_request_set $redirection_url $upstream_http_x_tinyauth_location;
    error_page 401 403 =302 $redirection_url;
  }
  location /tinyauth {
    internal;
    proxy_pass http://tinyauth:3000/api/auth/nginx;
    proxy_pass_request_body off;
    proxy_set_header content-length "";
    proxy_set_header x-original-url $scheme://$host$request_uri;
    proxy_set_header x-original-method $request_method;
    proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    proxy_set_header x-real-ip $remote_addr;
  }
  ```

- **Reverse proxy:** the Docker network `proxy` is created once per host (`docker network create proxy`; see `stacks/dockhand`). A stack attaches its web service to it and npm forwards by container name:

  ```yaml
  services:
    app:
      networks: [default, proxy]
  networks:
    proxy:
      external: true
  ```

  Then add a proxy host in npm (`app.casa.home` → `http://<container_name>:<port>`), with the Tinyauth block above unless the app keeps its own login.
- The media stack joins `proxy` entirely, so npm reaches `jellyfin`, `radarr`, `jellyseerr` and `gluetun` (qBittorrent) by container name.

## Backups (Backrest)

| Setting | Value |
| --- | --- |
| Repository | `homelab-local` → `/repos/homelab` = `${BACKUP_ROOT}/homelab` (an SSD, a different disk from `DATA_ROOT`) |
| Plan | `homelab-daily`: `/userdata/homelab-data` |
| Excludes | `homelab-data/backrest/cache`, `__pycache__`, and app log folders (`media/*/log`, `media/*/logs`, `media/qbittorrent/qBittorrent/logs`, `npm/logs`, `trek/data/logs`): about 60 MB that changes constantly and once made a run end "with warnings" when Radarr rotated a log mid-backup (2026-11-15) |
| Schedule | Every 24 h counted from the last run, so a backup runs soon after the PC is switched on |
| Retention | 24 hourly, 30 daily, 12 monthly |

- **The repository password lives outside this PC** (owner's password manager). Backrest's own config sits in `DATA_ROOT`; if that disk dies, the copies on the backup disk are unreadable without the password.
- Sources are mounted read-only. Restore to a writable path such as `/tmp/<name>` inside the container (then copy out), never over `/userdata`.
- Databases are copied live (no container stop). SQLite files restored fine in the test below; stop a stack before a backup used for migration.
- Only one copy, on the same PC. An offsite copy (cloud or external disk) is still pending.

**Restore test (2026-11-12):** snapshot `6c2d4363` (1.05 GiB source, 376 MB in the repository); `homelab-data` restored to `/tmp/restore-test`: 108/108 files, and `dockhand/.encryption_key`, `dockhand/db/dockhand.db`, `trek/data/travel.db`, `trek/data/.encryption_key`, `adguardhome/conf/AdGuardHome.yaml`, `npm/database.sqlite`, `npm/keys.json` byte-identical (sha256). Test copy deleted afterwards.

## Data

- `${DATA_ROOT}/adguardhome/` (DNS rules, rewrites, admin account), `${DATA_ROOT}/npm/` (proxy hosts, certificates, admin account), `${DATA_ROOT}/backrest/` (backup config).
- All of `DATA_ROOT` is backed up by the plan above.

## Checklist

- [x] Backup restore test of `DATA_ROOT` (files byte-identical, see Backups).
- [x] AdGuard + npm cutover completed 2026-11-12: DNS rewrites answer, `google.com` resolves upstream, all four npm proxy hosts reach the old media stack.
- [x] Registered and deployed as a Dockhand Git stack (2026-11-12). The hand-started project had to be removed first (`docker compose down`): Dockhand refuses a Git stack whose name matches an existing untracked one. Verified afterwards: AdGuard answers `jellyfin.home` and `google.com`, all four npm proxy hosts respond, Backrest keeps its plan and read-only mounts.
- [x] `*.home` names (2026-11-12): AdGuard wildcard rewrite added (any `<name>.home` answers the host IP); npm proxy hosts for all nine names in the table above respond through the host IP on port 80.
- [x] Shared login and `*.casa.home` (2026-11-13): Tinyauth deployed from Git; all 14 names moved; protected names redirect to `auth.casa.home`; one login opened the portal and the apps from a phone on the Wi-Fi. Own logins turned off in Radarr and Prowlarr (`AuthenticationMethod` `External` in `config.xml`, 2026-11-15: `DisabledForLocalAddresses` still asked for a login because npm forwards the client address), Bazarr (auth type null), qBittorrent (subnet whitelist `172.21.0.0/16`), AdGuard (no users) and Backrest (`auth.disabled`); each config was backed up as `*.bak-20261113-*` first. Uptime Kuma's own login was turned off by the owner in its settings (`disableAuth = true`).
- [x] DNS after a reboot (2026-11-13): AdGuard bound to the host IP (`DNS_BIND_IP`); LAN devices resolve again with ICS still running.
