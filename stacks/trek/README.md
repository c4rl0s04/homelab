# trek

[TREK](https://github.com/liketrek/TREK): self-hosted collaborative travel planner (AGPL-3.0). First stack deployed from Git through Dockhand.

One container, `trek`, on its own default network plus `proxy`. Upstream hardening is kept (read-only root filesystem, all capabilities dropped but four), and it is capped at 2 CPUs: on 2026-11-16 piled-up MCP sessions took 12 cores and starved every other stack.

## Deploy with Dockhand

1. **Stacks → Git stack**, repository `homelab`, branch `main`, compose file `stacks/trek/compose.yaml`. Dockhand reads it with the same GitHub token as the other stacks (see `stacks/dockhand/README.md`).
2. Add the stack variables from `.env.example`. `TREK_ENCRYPTION_KEY` is a secret: generate it once and keep it only in Dockhand.
3. **Create** only registers the stack ("no desplegado"). Press **Deploy** on its row and confirm the Git sync, then open `http://127.0.0.1:3002` and follow the first-login instructions below.

## Access

`http://trek.casa.home`, with TREK's own accounts (not behind Tinyauth). Over plain HTTP the session cookie must not be marked Secure, so `COOKIE_SECURE` defaults to `false`; set `TREK_COOKIE_SECURE=true` once the name is served over HTTPS.

**First login / locked out:** the first start creates an `admin` account (`admin@trek.local`) and prints a random password to the container log only once. To get in again, run `docker exec -it -u node trek node server/reset-admin.js`: it prints a temporary password and forces a new one at first login. Run it yourself; the password appears in the terminal.

**MCP:** From the host, connect to `http://localhost:3002/mcp` using OAuth. `TREK_APP_URL` defaults to `http://localhost` on the published port so OAuth discovery advertises an address the client can reach. If you override it, set it to the origin used by the MCP client and include that origin in `TREK_ALLOWED_ORIGINS` if you override that list too. Start with the `trips:read` scope to list trips and read itineraries.

## Data and backup

| Path | Content |
| --- | --- |
| `${DATA_ROOT}/trek/data` | SQLite database, JWT secret |
| `${DATA_ROOT}/trek/uploads` | Uploaded files and photos |

Back up both folders together with `TREK_ENCRYPTION_KEY` (stored in Dockhand). Stop the stack before copying for a consistent database.

## Restore

Copy both folders back to `${DATA_ROOT}/trek/`, set the same `TREK_ENCRYPTION_KEY`, deploy.

## Pilot checklist

- [x] Git stack deploys from `main` (commit `519afd2`, 2026-11-12); container healthy, `/api/health` ok, data under `${DATA_ROOT}/trek`.
- [x] `TREK_ENCRYPTION_KEY` reaches the container (64 chars) and appears nowhere in the repository or its history.
- [x] Change something in `compose.yaml`, push, sync in Dockhand: the change is applied (2026-11-12: joining the `proxy` network recreated the container).
- [x] Data survives redeploy (2026-11-12: after that recreation the container stayed healthy on the same database in `${DATA_ROOT}/trek` and the same `TREK_ENCRYPTION_KEY`).
