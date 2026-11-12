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
    npm["Nginx Proxy Manager<br/>*.home names"]
    backrest["Backrest<br/>backups"]
  end

  subgraph TREK["trek"]
    trek["TREK<br/>travel planner"]
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
  INFRA & TREK & MEDIA --> data
  MEDIA --> films[("films and downloads<br/>MEDIA_ROOT")]
  data -- every 24 h --> backups

  classDef manager fill:#ccfbf1,stroke:#0f766e,color:#042f2e
  classDef infra fill:#e0e7ff,stroke:#4338ca,color:#1e1b4b
  classDef trek fill:#fef3c7,stroke:#b45309,color:#451a03
  classDef media fill:#ffe4e6,stroke:#be123c,color:#4c0519
  classDef storage fill:#f1f5f9,stroke:#475569,color:#0f172a
  class dockhand manager
  class adguard,npm,backrest infra
  class trek trek
  class media media
  class git,data,backups,films storage
  style DOCKHAND fill:#f0fdfa,stroke:#0f766e,color:#042f2e
  style INFRA fill:#eef2ff,stroke:#4338ca,color:#1e1b4b
  style TREK fill:#fffbeb,stroke:#b45309,color:#451a03
  style MEDIA fill:#fff1f2,stroke:#be123c,color:#4c0519
```

One color per stack. Recipes live in Git, secrets in Dockhand, data in `homelab-data` (`DATA_ROOT`), copies in `homelab-backups` (`BACKUP_ROOT`). Network details (DNS, `*.home`, Docker networks) are in `stacks/infra/README.md`. Keep this diagram in sync with `stacks/` (see `AGENTS.md`).

## Layout

```text
stacks/
  dockhand/ # the stack manager; the only stack started by hand
  infra/    # AdGuard (DNS), Nginx Proxy Manager, Backrest; creates the `proxy` network; managed by Dockhand
  media/    # Jellyfin, Jellyseerr, Radarr, Prowlarr, Bazarr, qBittorrent behind ProtonVPN
  trek/     # TREK travel planner; first Git stack (pilot)
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
- Services are reached by hostname through the reverse proxy and local DNS, not `IP:port`.
- No Docker Desktop–only features (`host.docker.internal`, Windows paths, named pipes).
- One manager per stack: Dockhand manages everything except itself (`dockhand/`, by hand) and `paas/` (its own platform).
- Dockhand holds the Docker socket (root-equivalent): reachable only over LAN/Tailscale with authentication, never from the internet.

## Management

- **Dockhand** (`stacks/dockhand`) is the control panel. Each other stack is registered as a Git stack pointing at `stacks/<name>/compose.yaml`; host values and secrets are Dockhand stack variables. Change a Compose file, push, then sync/redeploy in Dockhand.
- **Homarr** is an optional later portal with links and read-only widgets.

## Moving to a new host

1. Install Docker. Stop every stack on the old host.
2. Copy the homelab folder (`repo`, `data`, `media`), or clone the repository and restore `data` from backup (it includes Dockhand's data and secret key).
3. Create `hosts/<host>.env` from `hosts/example.env` with the new paths.
4. Create the shared network once (`docker network create proxy`), start `stacks/dockhand` by hand; in Dockhand, update `DATA_ROOT` / `MEDIA_ROOT` / `BACKUP_ROOT` on each stack and deploy `infra` first, then the rest.
5. Repoint local DNS to the new host (router DHCP and the AdGuard `*.home` rewrite); verify every stack before retiring the old one.

A stack is migration-ready only after it has been restored from backup at least once.
