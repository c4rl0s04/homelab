# infra

Shared infrastructure used by every stack. **Managed by Dockhand as a Git stack** (the manager itself lives in `stacks/dockhand`). Stopping or redeploying this stack interrupts home DNS for a few seconds; the router's fallback resolver keeps Internet working meanwhile.

| Service | Purpose | Admin UI | Status |
| --- | --- | --- | --- |
| AdGuard Home | Home DNS: ad blocking and `*.home` names (DNS rewrites) for every device | `:${ADGUARD_UI_PORT:-3000}` | Running |
| Nginx Proxy Manager | Reverse proxy on 80/443: forwards `*.home` to each stack's web service | `:${NPM_ADMIN_PORT:-81}` | Running |
| Backrest | Encrypted, deduplicated backups (restic) of `DATA_ROOT` and the old media folder | `127.0.0.1:${BACKREST_PORT:-9898}` | Running |

Planned here later: Tailscale, Homarr (optional portal).

## Deploy with Dockhand

Git stack `infra`: repository `homelab`, branch `main`, compose file `stacks/infra/compose.yaml`. Stack variables (no secrets in this stack):

| Variable | Windows PC value |
| --- | --- |
| `DATA_ROOT` | `E:/homelab/data` |
| `BACKUP_ROOT` | `C:/homelab-backups` |
| `TZ` | `Europe/Madrid` |

Optional: `ADGUARD_UI_PORT` (3000), `NPM_ADMIN_PORT` (81), `BACKREST_PORT` (9898).

**Without Dockhand** (recovery, or a new host before Dockhand is up): `docker compose --env-file ../../hosts/<host>.env up -d` from this folder. Dockhand then shows the stack as untracked until it is registered again.

## How other stacks use it

- **DNS** works at network level: the router hands out this host as DNS server, and AdGuard rewrites `<app>.home` to the host IP. Keep a fallback resolver in the router (e.g. `1.1.1.1`) so the home network survives if this host is down.
- **Upstream DNS uses an IP literal** (`https://9.9.9.10/dns-query`, Quad9 unfiltered). On the Windows PC, ProtonVPN blocks plain DNS to anything but its own resolver, so AdGuard could not bootstrap `dns10.quad9.net` by name. The Windows host itself also cannot query AdGuard while ProtonVPN is connected; other devices can. Revisit on the mini PC (no desktop VPN there).
- **Router DHCP** (`192.168.1.1`) currently hands out `192.168.1.10, 1.1.1.1`. Devices may pick `1.1.1.1`, which does not know `*.home` (confirmed 2026-11-12: a phone works with DNS set manually to `192.168.1.10`). Kept on purpose while AdGuard runs on a PC that is not always on; remove `1.1.1.1` once the mini PC hosts DNS. Reserve the host's IP in the router so the DNS address never changes.
- **Port 53 on Windows:** the "Internet Connection Sharing" service (`SharedAccess`, used by Mobile Hotspot) can grab UDP 53 on IPv4; AdGuard then only gets IPv6 and LAN devices are answered by Windows instead. Stop it (`Stop-Service SharedAccess`, admin) and recreate `adguardhome`; disable it if the hotspot is not used.
- **Reverse proxy:** `infra` creates the Docker network `proxy`. A stack attaches only its web service to it and npm forwards by container name:

  ```yaml
  services:
    app:
      networks: [default, proxy]
  networks:
    proxy:
      external: true
  ```

  Then add a proxy host in npm (`app.home` → `http://<container_name>:<port>`) and a DNS rewrite in AdGuard if no wildcard covers it.
- The media stack joins `proxy` entirely, so npm reaches `jellyfin`, `radarr`, `jellyseerr` and `gluetun` (qBittorrent) by container name.

## Cutover from the old media stack (AdGuard + npm), done 2026-11-12

Kept as a record and rollback reference; it predates management by Dockhand.

Images are the same digests as before, so configuration loads unchanged. Old configuration is copied, not moved: it stays as rollback.

1. Stop the old stack (already stopped on 2026-11-12) and remove only its two containers. Bind-mounted config is untouched:

   ```bash
   docker rm adguardhome npm
   ```

2. Copy configuration into `DATA_ROOT`:

   ```bash
   robocopy <old-project>\configs\adguardhome <data-root>\adguardhome /E
   robocopy <old-project>\configs\npm <data-root>\npm /E
   ```

3. Start infra from `stacks/infra`: `docker compose --env-file ../../hosts/windows-pc.env up -d`.
4. Verify: `nslookup jellyfin.home 127.0.0.1` answers `192.168.1.10`; AdGuard UI on `:3000` and npm on `:81` keep their accounts, rules and proxy hosts.
5. Start the rest of the old media stack with `docker compose start` in `<old-project>`. **Never `docker compose up` there** until media migrates: it would recreate `adguardhome` and `npm` and clash on names and ports.
6. Check `jellyfin.home` and the others through npm.

**Rollback:** `docker compose --env-file ../../hosts/windows-pc.env rm -sf adguardhome npm` here, then `docker compose up -d adguardhome npm` in `<old-project>` (its original config was never changed).

## Backups (Backrest)

| Setting | Value |
| --- | --- |
| Repository | `homelab-local` → `/repos/homelab` = `${BACKUP_ROOT}/homelab` (`C:\homelab-backups\homelab`, SSD, different disk from `E:`) |
| Plan | `homelab-daily`: `/userdata/homelab-data` (the old `<old-project>` folder was dropped on 2026-11-12; earlier snapshots still hold it) |
| Excludes | `homelab-data/backrest/cache`, `__pycache__` |
| Schedule | Every 24 h counted from the last run, so a backup runs soon after the PC is switched on |
| Retention | 24 hourly, 30 daily, 12 monthly |

- **The repository password lives outside this PC** (owner's password manager). Backrest's own config sits in `DATA_ROOT` on `E:`; if that disk dies, the copies on `C:` are unreadable without the password.
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
