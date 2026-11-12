# dockhand

The stack manager: status, start/stop, statistics, logs, per-stack variables and secrets, and Git deployments for every other stack. **The only stack started by hand** — it must never manage itself (stopping or redeploying it from its own panel would remove the panel mid-operation).

## Start

On a new host, create the shared network once (every stack with a web UI joins it):

```bash
docker network create proxy
```

Then:

```bash
docker compose --env-file ../../hosts/<host>.env up -d
```

Open `http://dockhand.home` (or `http://127.0.0.1:${DOCKHAND_PORT:-3001}` from the host itself, which also works before `infra` is up).

## What it manages

| Stack | How |
| --- | --- |
| `infra` | Git stack from `stacks/infra/compose.yaml` |
| `trek` | Git stack from `stacks/trek/compose.yaml` |
| `streaming` (old media) | Observed only, untracked |

Git access uses a fine-grained GitHub token with read-only *Contents* access to this repository (Settings → Git → Credentials). Changes reach a stack only after they are pushed to `main` and the stack is deployed again.

## Data and backup

`${DATA_ROOT}/dockhand/` holds Dockhand's database, **including the encrypted stack variables and `.encryption_key`**. Without it, a migration loses every stack secret. It is part of the Backrest plan in `stacks/infra`.

## Checklist

- [x] Starts on Docker Desktop (2026-11-11, v1.0.57); container port `3000` and data path `/app/data` confirmed.
- [x] Local Docker environment added; authentication enabled.
- [x] Git stack deploys with an encrypted secret (`stacks/trek`, 2026-11-12).
- [ ] Change, push and sync an existing Git stack.
- [ ] Full restore: start Dockhand from a restored `${DATA_ROOT}/dockhand` and confirm stack secrets still decrypt.
