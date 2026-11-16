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

## Documentation

- The Mermaid diagram in `README.md` is the architecture map. Update it in the same change that adds, removes or moves a stack, a service, a published port, a Docker network or a data path. Keep it to what is actually deployed; planned stacks stay in the layout list only.

## Safety

- Secrets, host files (`hosts/*.env` except the example) and application data never enter Git.
- Pin image versions; no `latest`.
- Do not expose Dockhand or any admin UI to the internet.
- Validate Compose before committing: `docker compose -f stacks/<name>/compose.yaml --env-file hosts/example.env config --quiet`, adding `--env-file stacks/<name>/.env.example` for stacks that have one (`media`, `trek`).

## Git

- Never include the word `codex`, in any capitalization, in branch names or commit messages.
- Descriptive branch names (`feature/trek-stack`, `fix/...`, `docs/...`). Commit each small phase after its checks pass.
