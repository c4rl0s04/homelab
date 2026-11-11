# infra

Shared infrastructure. Started by hand, never managed from Dockhand (stopping it from its own panel would remove the panel).

| Service | Purpose | Status |
| --- | --- | --- |
| Dockhand | Control panel for all other stacks: status, start/stop, stats, logs, per-stack variables and secrets | Trial |

Planned here later: reverse proxy, local DNS, Tailscale, Homarr (optional portal), backup tool.

## Start

```bash
docker compose --env-file ../../hosts/<host>.env up -d
```

Open `http://127.0.0.1:${DOCKHAND_PORT}` (default `3000`; set `DOCKHAND_PORT` in the host file if that port is taken).

## Data and backup

- `${DATA_ROOT}/dockhand/` holds Dockhand's database, **including the encrypted stack variables and their key**. Without it, a migration loses every stack secret. Back it up with the rest of `DATA_ROOT`.

## Trial checklist

The old media stack (`streaming`, running from `<old-project>`) must stay untouched during the trial.

- [x] Dockhand starts on Docker Desktop (2026-11-11, v1.0.57); UI loads on the host port.
- [ ] Add the local Docker environment (socket) and confirm it lists the existing `streaming` containers.
- [ ] Enable authentication (Settings > Authentication) before any non-loopback access; it is off by default.
- [ ] Only observe `streaming`: no start/stop, redeploy or adoption.
- [x] Container port `3000` (image `EXPOSE`) and data path `/app/data` confirmed; the data directory holds `db/`, `git-repos/` and `.encryption_key`.
- [ ] Git stack test with a throwaway stack from this repo: deploy, change, sync, remove.
- [ ] Stack variables: set a test secret, confirm it reaches the container and is not visible in Git.
- [ ] Restore test: stop Dockhand, copy `${DATA_ROOT}/dockhand` elsewhere, start from the copy, secrets still work.
