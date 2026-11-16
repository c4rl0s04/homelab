# homelab

Git-versioned definition of a personal homelab made of independent Docker Compose stacks. Built to move from the current Windows PC to an always-on mini PC (Linux, or a Linux VM on Proxmox) with a routine procedure.

## Architecture

```mermaid
flowchart LR
  git[("GitHub<br/>stack recipes")]

  subgraph DOCKHAND["dockhand · started by hand"]
    dockhand["Dockhand<br/>control panel"]
  end

  subgraph INFRA["infra · shared by every stack"]
    adguard["AdGuard<br/>home DNS"]
    npm["Nginx Proxy Manager<br/>*.casa.home names"]
    tinyauth["Tinyauth<br/>one login"]
    backrest["Backrest<br/>backups"]
  end

  subgraph TREK["trek"]
    trek["TREK<br/>travel planner"]
  end

  subgraph MONITORING["monitoring"]
    kuma["Uptime Kuma<br/>alerts when a service is down"]
  end

  subgraph PORTAL["portal"]
    homepage["Homepage<br/>every app on one page"]
  end

  subgraph TOOLS["tools"]
    tools["PairDrop · Stirling-PDF<br/>file sharing and PDFs"]
  end

  subgraph MEDIA["media"]
    media["Jellyfin · Radarr<br/>qBittorrent · VPN"]
  end

  data[("homelab-data<br/>app data")]
  backups[("homelab-backups<br/>other disk")]

  git --> dockhand
  dockhand -- deploys --> INFRA
  dockhand -- deploys --> TREK
  dockhand -- deploys --> MEDIA
  dockhand -- deploys --> MONITORING
  dockhand -- deploys --> PORTAL
  dockhand -- deploys --> TOOLS
  INFRA & TREK & MEDIA & MONITORING & TOOLS --> data
  MEDIA --> films[("films and downloads<br/>MEDIA_ROOT")]
  data -- every 24 h --> backups

  classDef manager fill:#ccfbf1,stroke:#0f766e,color:#042f2e
  classDef infra fill:#e0e7ff,stroke:#4338ca,color:#1e1b4b
  classDef trek fill:#fef3c7,stroke:#b45309,color:#451a03
  classDef media fill:#ffe4e6,stroke:#be123c,color:#4c0519
  classDef monitoring fill:#dcfce7,stroke:#15803d,color:#052e16
  classDef portal fill:#e0f2fe,stroke:#0369a1,color:#082f49
  classDef tools fill:#f3e8ff,stroke:#7e22ce,color:#3b0764
  classDef storage fill:#f1f5f9,stroke:#475569,color:#0f172a
  class dockhand manager
  class adguard,npm,backrest,tinyauth infra
  class trek trek
  class media media
  class kuma monitoring
  class homepage portal
  class tools tools
  class git,data,backups,films storage
  style DOCKHAND fill:#f0fdfa,stroke:#0f766e,color:#042f2e
  style INFRA fill:#eef2ff,stroke:#4338ca,color:#1e1b4b
  style TREK fill:#fffbeb,stroke:#b45309,color:#451a03
  style MEDIA fill:#fff1f2,stroke:#be123c,color:#4c0519
  style MONITORING fill:#f0fdf4,stroke:#15803d,color:#052e16
  style PORTAL fill:#f0f9ff,stroke:#0369a1,color:#082f49
  style TOOLS fill:#faf5ff,stroke:#7e22ce,color:#3b0764
```

One color per stack. Recipes live in Git, secrets in Dockhand, data in `homelab-data` (`DATA_ROOT`), copies in `homelab-backups` (`BACKUP_ROOT`). Network details (DNS, `*.casa.home`, the shared login, Docker networks) are in `stacks/infra/README.md`. Keep this diagram in sync with `stacks/` (see `AGENTS.md`).

## Layout

```text
stacks/
  dockhand/ # the stack manager; the only stack started by hand
  infra/    # AdGuard (DNS), Nginx Proxy Manager, Backrest; creates the `proxy` network; managed by Dockhand
  media/    # Jellyfin, Jellyseerr, Radarr, Prowlarr, Bazarr, qBittorrent behind ProtonVPN
  trek/     # TREK travel planner; first Git stack (pilot)
  monitoring/ # Uptime Kuma: checks every service, separate so it survives other stacks' redeploys
  portal/   # Homepage: every app on one page; its configuration is inline in compose.yaml
  tools/    # PairDrop (send files between devices) and Stirling-PDF (PDF toolbox)
  photos/   # planned: Immich candidate
  paas/     # planned: Coolify/Dokploy, on its own host/VM, not managed by Dockhand
hosts/
  example.env   # template for host-specific values; real files stay private
```

On the host everything lives under one folder, so moving the homelab means copying it:

```text
E:\homelab\            (/srv/homelab on Linux)
  repo\     # this repository
  data\     # DATA_ROOT: app data, Dockhand's encrypted secrets
  media\    # MEDIA_ROOT: downloads/ and media/ (films)
  snapshot-worktree\  # scratch checkout used by scripts/snapshot-config.sh
C:\homelab-backups\    # BACKUP_ROOT: deliberately on another disk
```

Each stack folder holds:

- `compose.yaml`: pinned image versions, variables only, no secrets, no host paths.
- `.env.example`: variable names with safe example values (when the stack needs any).
- `README.md`: what data it owns and how to back it up and restore it.

## Layers

| Layer | Location | In Git |
| --- | --- | --- |
| Definition | `stacks/<name>/` | Yes |
| Secrets | Dockhand per-stack variables (encrypted in its database); host file only for `dockhand/` | Never |
| Application data | `${DATA_ROOT}/<stack>/` bind mounts | No; per-stack backups |
| Bulk media | `${MEDIA_ROOT}` on a separate disk | No; moved or copied directly |

## Rules

- Paths, `PUID`/`PGID` and `TZ` come from the host file. A missing variable fails (`${VAR:?...}`) instead of falling back to a path.
- Bind mounts under `DATA_ROOT`, not anonymous volumes: copying a folder copies a stack's state.
- Every Compose file sets `name:` so the project name does not depend on its folder.
- Every service caps its logs (`logging: *logging`, 3 x 10 MB): Docker keeps container logs forever by default.
- Services that can spike (TREK, Neko, Byparr) get `cpus: 2`: on 2026-11-16 TREK took 12 cores and every other stack, npm and Tinyauth included, stopped answering.
- Services are reached by hostname through the reverse proxy and local DNS, not `IP:port`. Web UIs protected by Tinyauth publish their port on `127.0.0.1` only, so the LAN cannot bypass the login.
- No Docker Desktop–only features (`host.docker.internal`, Windows paths, named pipes).
- One manager per stack: Dockhand manages everything except itself (`dockhand/`, by hand) and `paas/` (its own platform).
- Dockhand holds the Docker socket (root-equivalent): reachable only over LAN/Tailscale with authentication, never from the internet.

## Management

- **Dockhand** (`stacks/dockhand`) is the control panel. Each other stack is registered as a Git stack pointing at `stacks/<name>/compose.yaml`; host values and secrets are Dockhand stack variables. Change a Compose file, push, then sync/redeploy in Dockhand.
- **Portal** (`stacks/portal`, `portal.casa.home`): Homepage, configured in Git. Read-only: links, status and widgets; it never changes a stack.

## Roadmap

Planned, not deployed. Each becomes its own stack when started.

| Item | Status | Waits for |
| --- | --- | --- |
| Vaultwarden (`vault`), homelab passwords only | Planned | HTTPS: Bitwarden clients refuse plain HTTP; serve it through Tailscale HTTPS or a trusted local certificate |
| Tailscale (remote access to `*.casa.home`) | Client installed on the Windows host | Subnet route or DNS setup |
| Immich (`photos`) | Postponed | Photo storage location and an offsite backup (photos are irreplaceable) |
| Home Assistant (`home`) + extras (Mushroom, Bubble Card, Adaptive Lighting, Powercalc) | Postponed | Mini PC: needs host networking for device discovery (WiZ, HomeKit), which Docker Desktop on Windows does not give |
| Zigbee2MQTT | Only if Zigbee devices are bought | Home Assistant and a USB Zigbee adapter |
| Offsite backup copy | Pending | Destination choice |

## Configuration snapshots

Some settings live in app databases, not in Git: npm proxy hosts (and their Tinyauth block), AdGuard rewrites and upstreams, Uptime Kuma monitors, each app's login mode and Dockhand's stack variables. `snapshots/*.json` records them so they can be redone by hand after a rebuild.

- `scripts/snapshot_config.py <DATA_ROOT>` reads the app files read-only and writes the JSON. Secret values are replaced by `<secret>` (Dockhand secrets and any variable named `*KEY*`, `*TOKEN*`, `*PASSWORD*`, `*SECRET*`); the script refuses to write output that still looks like a key or hash.
- `scripts/snapshot-config.sh hosts/<host>.env` runs it in a separate worktree (`../snapshot-worktree`) and opens a PR "Configuration snapshot <date>" only when something changed. Run it daily and at logon (Windows: Task Scheduler; Linux: a systemd timer or cron).
- Snapshots are a record, not a source of truth: restoring means re-entering these values in each app.

## Moving to a new host

1. Install Docker. Stop every stack on the old host.
2. Copy the homelab folder (`repo`, `data`, `media`), or clone the repository and restore `data` from backup (it includes Dockhand's data and secret key).
3. Create `hosts/<host>.env` from `hosts/example.env` with the new paths.
4. Create the shared network once (`docker network create proxy`), start `stacks/dockhand` by hand; in Dockhand, update `DATA_ROOT` / `MEDIA_ROOT` / `BACKUP_ROOT` on each stack and deploy `infra` first, then the rest.
5. Repoint local DNS to the new host (router DHCP and the AdGuard `*.home` rewrite, which covers `*.casa.home`); verify every stack before retiring the old one.

A stack is migration-ready only after it has been restored from backup at least once.
