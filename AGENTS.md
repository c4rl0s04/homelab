# Repository working conventions

Read `README.md` first: it defines the stack layout, layers and portability rules. Every new stack must follow them.

## Current state

- `stacks/dockhand` (the manager, started by hand) deploys `stacks/infra` (AdGuard, Nginx Proxy Manager, Backrest) and `stacks/trek` as Git stacks. See each stack's checklist before relying on Dockhand behaviour.
- The live media stack still runs from the previous repository (`c4rl0s04/previous-project`, local `<old-project>`, Compose project `streaming`). Do not stop, adopt, redeploy or modify it from here. It moves to `stacks/media` only through a planned cutover: stop old stack, copy its `configs/` into `${DATA_ROOT}/media`, start the new stack, verify, keep the old one as rollback.
- No custom control panel or orchestration code: use Dockhand, the native application UIs and standard backup tooling.

## Safety

- Secrets, host files (`hosts/*.env` except the example) and application data never enter Git.
- Pin image versions; no `latest`.
- Do not expose Dockhand or any admin UI to the internet.
- Validate Compose before committing: `docker compose -f stacks/<name>/compose.yaml --env-file hosts/example.env config --quiet`.

## Git

- Never include the word `codex`, in any capitalization, in branch names or commit messages.
- Descriptive branch names (`feature/trek-stack`, `fix/...`, `docs/...`). Commit each small phase after its checks pass.
