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

Open `http://dockhand.casa.home` (Tinyauth, then Dockhand's own login) (or `http://127.0.0.1:${DOCKHAND_PORT:-3001}` from the host itself, which also works before `infra` is up).

## What it manages

| Stack | How |
| --- | --- |
| `infra` | Git stack from `stacks/infra/compose.yaml` |
| `media` | Git stack from `stacks/media/compose.yaml` |
| `trek` | Git stack from `stacks/trek/compose.yaml` |
| `monitoring` | Git stack from `stacks/monitoring/compose.yaml` |
| `portal` | Git stack from `stacks/portal/compose.yaml` |
| `tools` | Git stack from `stacks/tools/compose.yaml` |

The old `streaming` stack was replaced by `media` on 2026-11-12. Do not start both: their container names and ports overlap.

Git access uses a fine-grained GitHub token with read-only *Contents* access to this repository (Settings → Git → Credentials). Changes reach a stack only after they are pushed to `main` and the stack is deployed again.

Its own variables come from `hosts/<host>.env` (`DATA_ROOT`, `TZ`, optional `DOCKHAND_PORT`), not from Dockhand.

## Applying a change

The UI is in Spanish. On a stack's row:

| Change | Action |
| --- | --- |
| `compose.yaml` services, images, ports, volumes | Push to `main`, then **Sincronizar desde Git** (pulls and redeploys) or **Desplegar** and confirm the Git sync |
| A stack variable or secret | Edit it on the stack, then **Desplegar**: containers only see new values when recreated |
| Inline `configs` content only (`portal`) | **Desmontar**, then **Desplegar**: Compose does not recreate a container when only a config's content changed |
| Turn on an optional service (a Compose profile, e.g. Neko in `tools`) | Add the stack variable `COMPOSE_PROFILES=<profile>` and deploy. Removing it and deploying again does not stop the running container: `docker rm -f <name>` |

A Git stack whose name matches a project started by hand is refused: `docker compose down` that project first. Stacks stay in Dockhand when their containers are stopped; deploy them again to start them.

## Start order after a full stop

Every container uses `restart: unless-stopped`, so a container stopped by hand stays stopped after a reboot. To bring everything back:

1. `infra` (AdGuard, npm, Tinyauth, Backrest): names, proxy and login.
2. `dockhand` (`docker start dockhand`, or `docker compose up -d` here).
3. The rest in any order: `monitoring`, `media`, `portal`, `tools`, `trek`. Either deploy them from Dockhand or `docker start` their containers.

## Data and backup

`${DATA_ROOT}/dockhand/` holds Dockhand's database, **including the encrypted stack variables and `.encryption_key`**. Without it, a migration loses every stack secret. It is part of the Backrest plan in `stacks/infra`.

## Checklist

- [x] Starts on Docker Desktop (2026-11-11, v1.0.57); container port `3000` and data path `/app/data` confirmed.
- [x] Local Docker environment added; authentication enabled.
- [x] Git stack deploys with an encrypted secret (`stacks/trek`, 2026-11-12).
- [x] Change, push and sync an existing Git stack (2026-11-12, `monitoring`: new setting applied, data kept). Variable changes need a deploy; inline `configs` changes need Desmontar + Desplegar (`portal`).
- [ ] Full restore: start Dockhand from a restored `${DATA_ROOT}/dockhand` and confirm stack secrets still decrypt.
