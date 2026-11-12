# infra

Shared infrastructure used by every stack. Started by hand, never managed from Dockhand (stopping it from its own panel would remove the panel).

| Service | Purpose | Admin UI | Status |
| --- | --- | --- | --- |
| Dockhand | Control panel for all other stacks | `127.0.0.1:${DOCKHAND_PORT:-3001}` | Trial |
| AdGuard Home | Home DNS: ad blocking and `*.home` names (DNS rewrites) for every device | `:${ADGUARD_UI_PORT:-3000}` | Moving from the old media stack |
| Nginx Proxy Manager | Reverse proxy on 80/443: forwards `*.home` to each stack's web service | `:${NPM_ADMIN_PORT:-81}` | Moving from the old media stack |

Planned here later: Tailscale, Homarr (optional portal), backup tool.

## Start

```bash
docker compose --env-file ../../hosts/<host>.env up -d
```

## How other stacks use it

- **DNS** works at network level: the router hands out this host as DNS server, and AdGuard rewrites `<app>.home` to the host IP. Keep a fallback resolver in the router (e.g. `1.1.1.1`) so the home network survives if this host is down.
- **Upstream DNS uses an IP literal** (`https://9.9.9.10/dns-query`, Quad9 unfiltered). On the Windows PC, ProtonVPN blocks plain DNS to anything but its own resolver, so AdGuard could not bootstrap `dns10.quad9.net` by name. The Windows host itself also cannot query AdGuard while ProtonVPN is connected; other devices can. Revisit on the mini PC (no desktop VPN there).
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
- **Transitional:** npm also joins `streaming_default` to reach the old media stack (`jellyfin`, `radarr`, `gluetun`, `jellyseerr`). That network must exist (the old stack stopped is fine; `docker compose down` there would remove it and block `infra`). Remove it once media moves to `stacks/media`.

## Cutover from the old media stack (AdGuard + npm)

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

## Data and backup

- `${DATA_ROOT}/dockhand/` holds Dockhand's database, **including the encrypted stack variables and their key**. Without it, a migration loses every stack secret.
- `${DATA_ROOT}/adguardhome/` (DNS rules, rewrites, admin account) and `${DATA_ROOT}/npm/` (proxy hosts, certificates, admin account).
- Back up all of `DATA_ROOT`.

## Trial checklist

- [x] Dockhand starts on Docker Desktop (2026-11-11, v1.0.57); UI loads on the host port.
- [x] Local Docker environment added; lists the existing `streaming` stack as untracked.
- [x] Authentication enabled.
- [x] Container port `3000` (image `EXPOSE`) and data path `/app/data` confirmed; the data directory holds `db/`, `git-repos/` and `.encryption_key`.
- [x] Git stack deploys with an encrypted secret (`stacks/trek`, 2026-11-12).
- [ ] Change, push and sync an existing Git stack.
- [ ] Restore test: stop Dockhand, copy `${DATA_ROOT}/dockhand` elsewhere, start from the copy, secrets still work.
- [x] AdGuard + npm cutover completed 2026-11-12: DNS rewrites answer, `google.com` resolves upstream, all four npm proxy hosts reach the old media stack, media started with `docker compose start`.
