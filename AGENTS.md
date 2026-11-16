# Repository working conventions

Read `README.md` first: it defines the stack layout, layers and portability rules. Every new stack must follow them.

## Current state

- `stacks/dockhand` (the manager, started by hand) deploys every other stack as a Git stack: `infra` (AdGuard, Nginx Proxy Manager, Tinyauth, Backrest), `media`, `trek`, `monitoring` (Uptime Kuma, Beszel), `portal` (Homepage) and `tools` (PairDrop, Stirling-PDF, Neko). See each stack's checklist before relying on Dockhand behaviour.
- Applying changes: push, then sync the stack in Dockhand. New or changed stack variables only reach containers on deploy. `portal` needs Desmontar + Desplegar for config changes (see its README).
- Names are `<app>.casa.home`; most web UIs sit behind Tinyauth (one login) with their own login off and their port on `127.0.0.1`. See the name table in `stacks/infra/README.md` before adding or exposing a web UI.
- Planned work and its prerequisites: the roadmap in `README.md`.
- On the current host, the repository, data and media share one folder (`repo`, `data`, `media`); backups sit on another disk. Real paths live in Dockhand variables and `hosts/<host>.env`, never in Git.
- `stacks/media` replaced an older single Compose project on 2026-11-12; its containers share names and ports with `media`, so never start both.
- No custom control panel or orchestration code: use Dockhand, the native application UIs and standard backup tooling.
- The current host is a Windows 11 PC running Docker Desktop (WSL2 VM with 8 GB of memory and 8 CPUs). On Docker Desktop, "the host" seen from containers and by Beszel is that VM, not Windows. A Linux mini PC is planned.
- How to apply each kind of change, and the start order after a full stop: `stacks/dockhand/README.md`.

## State that lives outside Git

Compose files are only part of the homelab. This state is kept in each app's data under `DATA_ROOT` (and backed up by Backrest), so a change to it is made in that app's UI or API and documented in the owning README:

| State | Lives in | Documented in |
| --- | --- | --- |
| Proxy hosts and their Advanced (Tinyauth) blocks | Nginx Proxy Manager database | Name table and Tinyauth block in `stacks/infra/README.md`; the Beszel extra lines in `stacks/monitoring/README.md` |
| The `*.home` DNS rewrite and the upstream resolver | AdGuard config | `stacks/infra/README.md` |
| Tinyauth users | `${DATA_ROOT}/tinyauth/users.txt` | `stacks/infra/README.md` |
| Backup repository, plan, excludes and schedule | Backrest config | `stacks/infra/README.md` |
| Stack variables and secrets | Dockhand database (encrypted) | Each stack's variable table |
| Monitors and the `homelab` status page | Uptime Kuma database | `stacks/monitoring/README.md` |
| Links between the media apps (download client, indexers, paths) | Each app's database | `stacks/media/README.md` |
| Own logins turned off behind Tinyauth | Each app's config | `stacks/infra/README.md` checklist |

## Adding a web service

1. The service in its stack's `compose.yaml`: pinned image, log cap, port on `127.0.0.1`, joined to `proxy`, data under `${DATA_ROOT}/<app>`.
2. An npm proxy host `<app>.casa.home` → `<container>:<port>`, with the Tinyauth block unless it keeps its own login. Turn the app's own login off when it is behind Tinyauth. DNS needs nothing: the wildcard rewrite covers new names.
3. A row in the name table in `stacks/infra/README.md`.
4. An Uptime Kuma monitor by container name, added to the `homelab` status page.
5. A portal card in `stacks/portal/compose.yaml` and its `INFO` entry in `custom-js`.
6. The architecture map and its PNGs (see Documentation), and the root README if it is user-visible.
7. Backups need nothing if its data is under `DATA_ROOT`; exclude log or cache folders that change constantly.

## Documentation

- `docs/architecture.html` is the architecture map (self-contained, logos inlined); `docs/architecture.png` and `docs/architecture-dark.png` are its screenshots used by `README.md`. Update it in the same change that adds, removes or moves a stack or a service, then re-render both PNGs (headless Chromium/Edge, window 1640x1250, scale 2; dark = `data-theme="dark"` on `<html>`). Keep it to what is actually deployed; planned items stay in the README roadmap.
- `README.md` is the public showcase; operator detail belongs in each stack's README. Screenshots live in `docs/screens/`; retake the affected one when a UI changes visibly, and check it for private data (usernames, IPs, visited domains) before committing.

## Safety

- Secrets, host files (`hosts/*.env` except the example) and application data never enter Git.
- Pin image versions; no `latest`.
- Do not expose Dockhand or any admin UI to the internet.
- Validate Compose before committing: `docker compose -f stacks/<name>/compose.yaml --env-file hosts/example.env config --quiet`, adding `--env-file stacks/<name>/.env.example` for stacks that have one (`media`, `trek`).

## Git

- Never include the word `codex`, in any capitalization, in branch names or commit messages.
- Descriptive branch names (`feature/trek-stack`, `fix/...`, `docs/...`). Commit each small phase after its checks pass.
