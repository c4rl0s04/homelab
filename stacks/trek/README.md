# trek

[TREK](https://github.com/mauriceboe/TREK): self-hosted collaborative travel planner (AGPL-3.0). First stack deployed from Git through Dockhand.

## Deploy with Dockhand

1. **Stacks → Git stack**, repository `c4rl0s04/homelab`, branch `main`, compose file `stacks/trek/compose.yaml`. The repository is private: give Dockhand a fine-grained GitHub token with read-only *Contents* access to this repository only.
2. Add the stack variables from `.env.example`. `TREK_ENCRYPTION_KEY` is a secret: generate it once and keep it only in Dockhand.
3. Deploy, then open `http://127.0.0.1:3002` and create the first account (it becomes the admin).

## Data and backup

| Path | Content |
| --- | --- |
| `${DATA_ROOT}/trek/data` | SQLite database, JWT secret |
| `${DATA_ROOT}/trek/uploads` | Uploaded files and photos |

Back up both folders together with `TREK_ENCRYPTION_KEY` (stored in Dockhand). Stop the stack before copying for a consistent database.

## Restore

Copy both folders back to `${DATA_ROOT}/trek/`, set the same `TREK_ENCRYPTION_KEY`, deploy.

## Pilot checklist

- [ ] Git stack deploys from `main`; container becomes healthy.
- [ ] `TREK_ENCRYPTION_KEY` reaches the container and appears nowhere in Git.
- [ ] Change something in `compose.yaml`, push, sync in Dockhand: the change is applied.
- [ ] Data survives stop/start and redeploy.
