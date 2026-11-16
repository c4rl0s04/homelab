# monitoring

[Uptime Kuma](https://github.com/louislam/uptime-kuma): checks every web service and alerts when one stops answering.

It also runs [Beszel](https://github.com/henrygd/beszel): history of CPU, memory, network, disk and GPU for the host and each container. Kuma answers "is it up?"; Beszel answers "what is using the machine?".

Its own stack on purpose: it must keep running while the stacks it watches (including `infra`) are stopped or redeployed.

## Deploy with Dockhand

Git stack `monitoring`: repository `homelab`, branch `main`, compose file `stacks/monitoring/compose.yaml`. Stack variables (store the agent key and token as secrets):

| Variable | Windows PC value |
| --- | --- |
| `DATA_ROOT` | `E:/homelab/data` |
| `TZ` | `Europe/Madrid` |
| `BESZEL_AGENT_KEY` | Secret, from Beszel's "Add system" |
| `BESZEL_AGENT_TOKEN` | Secret, from Beszel's "Add system" |

Optional: `UPTIME_KUMA_PORT` (3003), `BESZEL_PORT` (3007), `PROXY_SUBNET` (`172.21.0.0/16`, the `proxy` network). After deploying, open `http://status.casa.home` (npm proxy host → `uptime-kuma:3001`, websockets on, behind Tinyauth; its own login is disabled).

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
| Neko | `http://neko:8080` |
| Beszel | `http://beszel:8090/api/health` |

One DNS monitor resolves `jellyfin.casa.home` against the host's LAN IP to cover AdGuard itself. It goes through the Windows host network, so it fails while ProtonVPN is connected on the host (which blocks DNS to other servers); LAN devices are not affected. Expect it to recover on the mini PC.

## Beszel

Three containers:

- `beszel` is the hub, the web page at `beszel.casa.home`.
- `beszel-agent` collects the numbers. It uses the NVIDIA image and is given the GPU, which works on Docker Desktop with WSL2.
- `beszel-socket-proxy` gives the agent read-only access to the container list and stats, so the agent never touches the Docker socket itself.

The agent and the proxy sit on an internal network with no internet access. On Docker Desktop the "host" Beszel sees is Docker's Linux VM, not Windows itself.

**First start**

1. Deploy, then open `beszel.casa.home` and create the account with email `<user>@casa.home`, where `<user>` is your Tinyauth username in lowercase: that is the address Tinyauth sends (username at the cookie domain).
2. In Beszel, click **Add system**: name `windows-pc`, host `beszel-agent`. Copy the **Key** and **Token** into Dockhand as secrets `BESZEL_AGENT_KEY` and `BESZEL_AGENT_TOKEN`, then redeploy `monitoring`. Until then the agent exits and retries.

**One login**

npm forwards Tinyauth's `Remote-Email` header to Beszel, which logs in the user with that email. Beszel trusts the header only from the `proxy` network (`TRUSTED_PROXY_IPS`), so opening the loopback port directly still asks for a password.

## Status page

Status page `homelab` (slug) lists every monitor; the portal's Uptime Kuma widget reads it (`/status/homelab`). Add new monitors to it, or the widget will not count them.

## Data and backup

`${DATA_ROOT}/uptime-kuma` holds monitors, history and account data; `${DATA_ROOT}/beszel` holds the Beszel hub data. Both are included in the documented `DATA_ROOT` backup plan. Agent credentials are stored in Dockhand.

## Checklist

- [x] Deployed as a Git stack (2026-11-12); `status.home` answers through npm. Database: SQLite (chosen at first start).
- [x] Monitors for every service above, plus Prowlarr, Bazarr and the DNS check, are green (12/12, 2026-11-12). Two retries before a service counts as down.
- [x] Change → push → sync test (2026-11-12): `stop_grace_period: 30s` merged in PR #9, synced in Dockhand; the container was recreated with `StopTimeout=30` and all 12 monitors and their history survived.
- [x] Beszel (2026-11-16): hub, agent and socket proxy deployed; the agent connects over WebSocket and reports CPU, memory, disk and the RTX 2070 GPU. `beszel.casa.home` redirects to Tinyauth; a forged `Remote-Email` from outside npm is ignored; port 3007 is not reachable from the LAN. Monitors for Neko and Beszel added and on the `homelab` status page (17 monitors).
- [ ] Owner opens `beszel.casa.home` through Tinyauth and lands logged in, with no Beszel password.
