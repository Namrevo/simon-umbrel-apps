# BookOrbit for Umbrel (v2)

This version is built directly from BookOrbit's real `docker-compose.yml` and
`.env.example` (fetched straight from the repo), adapted for Umbrel's app
model. Two things went wrong before, both worth knowing about:

- My first package guessed at the image and env vars without the real
  compose file in hand, so some values didn't match what the app actually
  expects.
- BookOrbit's **official** compose file is written for a plain Docker host,
  not Umbrel, so running it as-is breaks in three specific ways:
  1. It publishes a host port directly (`ports: "3000:3000"`) — Umbrel
     expects `app_proxy` to own port exposure, not the app container.
  2. It loads secrets from a `.env` file via `env_file: - .env` — Umbrel
     doesn't create that file for you, so `POSTGRES_PASSWORD`, `JWT_SECRET`,
     etc. (all marked `?required`) are simply missing and the container
     exits.
  3. It bind-mounts relative host paths (`./books`, `./data/app`,
     `./data/postgres`) — under Umbrel these don't resolve to your app's
     actual data directory, so the app and Postgres can end up reading and
     writing in the wrong (or an inaccessible) place.

This package fixes all three: `app_proxy` handles the port, every secret is
set directly in `docker-compose.yml` using Umbrel's per-app secrets
(`$APP_SEED`, `$APP_PASSWORD`), and volumes are anchored under
`${APP_DATA_DIR}`. Everything else — the health check, read-only root
filesystem, dropped capabilities — is carried over unchanged from the real
compose file.

## Before you reinstall

Uninstall the old broken install first and clear out its data directory, so
Postgres doesn't try to start against a half-initialized database from the
earlier attempt:

```bash
sudo ~/umbrel/scripts/app uninstall custom-bookorbit
sudo rm -rf ~/umbrel/app-data/custom-bookorbit
```

(Skip this if you never got as far as installing it, or if you don't mind
losing whatever data the earlier attempt wrote.)

## Publish and install

Same as before — if you already have the repo from last time, just replace
`custom-bookorbit/docker-compose.yml` with the new one and push:

```bash
cd bookorbit-umbrel-app-store
git add .
git commit -m "Fix docker-compose.yml for Umbrel"
git push
```

Then on Umbrel:

```bash
sudo ~/umbrel/scripts/repo update
sudo ~/umbrel/scripts/app install custom-bookorbit
```

(Or via the dashboard: App Store → your custom store → Install. If Umbrel
already has an old cached copy, use "Update" instead, or uninstall/reinstall
if the tile is stuck.)

Give Postgres 20–30 seconds to finish initializing before opening the app.

## Complete setup

Open the app from the dashboard, then grab the one-time setup token over
SSH:

```bash
grep -A1 SETUP_BOOTSTRAP_TOKEN ~/umbrel/app-data/custom-bookorbit/docker-compose.yml
```

Paste it into the setup screen and create your admin account.

## If it still won't start

Check the container logs — this will usually say exactly what's wrong
(bad env var, permission error, Postgres not ready yet):

```bash
sudo ~/umbrel/scripts/app compose custom-bookorbit logs app
sudo ~/umbrel/scripts/app compose custom-bookorbit logs postgres
```

Common culprits at this stage: the two containers not both running (check
`sudo ~/umbrel/scripts/app compose custom-bookorbit ps`), or leftover data
from a previous attempt with different Postgres credentials than the ones
Umbrel is now generating — the "before you reinstall" step above clears
that.

## Notes

- **Image tag**: pins `ghcr.io/bookorbit/bookorbit:latest`. BookOrbit's own
  `.env.example` recommends pinning a specific `sha-*` tag or digest for
  reproducible deploys — check
  https://github.com/bookorbit/bookorbit/pkgs/container/bookorbit for
  available tags if you want that.
- **External database**: BookOrbit supports pointing at a Postgres you
  already run via `DATABASE_URL` instead of the bundled container (needs
  the `uuid-ossp`, `pg_trgm`, and `vector` extensions). Drop the `postgres:`
  service and add `DATABASE_URL` to the `app` service's environment if you
  want that instead.
- **Books folder**: books live at
  `~/umbrel/app-data/custom-bookorbit/data/books`. Point that volume line at
  a different host path if your library already lives elsewhere (e.g. a
  network share mounted into Umbrel).
- **Icon/gallery**: still just cosmetic placeholders — see the previous
  README section on this if you want to add them.
