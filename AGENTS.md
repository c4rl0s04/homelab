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

- `docs/architecture.html` is the architecture map (self-contained, logos inlined); `docs/architecture.png` and `docs/architecture-dark.png` are its screenshots used by `README.md`. Update it in the same change that adds, removes or moves a stack or a service, then re-render both PNGs (headless Chromium/Edge, window 1640x1250, scale 2; dark = `data-theme="dark"` on `<html>`). Keep it to what is actually deployed; planned items stay in the README roadmap.
- `README.md` is the public showcase; operator detail belongs in each stack's README. Retake the portal and status screenshots in `docs/` when the portal changes visibly.

## Safety

- Secrets, host files (`hosts/*.env` except the example) and application data never enter Git.
- Pin image versions; no `latest`.
- Do not expose Dockhand or any admin UI to the internet.
- Validate Compose before committing: `docker compose -f stacks/<name>/compose.yaml --env-file hosts/example.env config --quiet`, adding `--env-file stacks/<name>/.env.example` for stacks that have one (`media`, `trek`).

## Git

- Never include the word `codex`, in any capitalization, in branch names or commit messages.
- Descriptive branch names (`feature/trek-stack`, `fix/...`, `docs/...`). Commit each small phase after its checks pass.
