# BookOrbit for Umbrel

This is a small "Community App Store" containing one app — BookOrbit — packaged
to run on Umbrel Home the same way apps from the official App Store do (its
own dashboard tile, `https://umbrel.local` access via the app proxy, and data
kept under Umbrel's normal app-data directory). BookOrbit isn't in the
official Umbrel App Store yet, so this is the standard way to add an app
Umbrel doesn't ship: https://github.com/getumbrel/umbrel-community-app-store

## What's inside

```
umbrel-app-store.yml          <- declares this as a community app store
custom-bookorbit/
  umbrel-app.yml               <- app manifest (name, version, description)
  docker-compose.yml           <- BookOrbit + bundled Postgres/pgvector
```

The compose file mirrors BookOrbit's official install (from
https://bookorbit.app/installation): a `web` container plus a `postgres`
container using `pgvector/pgvector:pg18`, with `/books` and app data mounted
under Umbrel's per-app data directory.

## 1. Publish it somewhere Umbrel can reach

Umbrel adds community stores by git URL, so push this folder to a repo (a
private GitHub repo works fine — Umbrel only needs to `git clone` it):

```bash
cd bookorbit-umbrel-app-store
git init
git add .
git commit -m "BookOrbit app for Umbrel"
git remote add origin https://github.com/<your-username>/umbrel-bookorbit-store.git
git push -u origin main
```

## 2. Add the store to Umbrel

**From the dashboard:** open the App Store → the "⋯" menu → **Add a Community
App Store** → paste your repo URL.

**Or via SSH**, if you'd rather use the CLI:

```bash
sudo ~/umbrel/scripts/repo add https://github.com/<your-username>/umbrel-bookorbit-store.git
sudo ~/umbrel/scripts/repo update
```

## 3. Install BookOrbit

It'll show up under "My Custom App Store" in the dashboard — click Install.
Or from SSH:

```bash
sudo ~/umbrel/scripts/app install custom-bookorbit
```

Give it 20–30 seconds on first start for Postgres to initialize.

## 4. Complete setup

Open the app from your Umbrel dashboard. It'll ask for a **setup bootstrap
token**. Umbrel generates a unique secret per app rather than letting a
compose file hard-code one, so grab it over SSH:

```bash
grep SETUP_BOOTSTRAP_TOKEN ~/umbrel/app-data/custom-bookorbit/docker-compose.yml
```

Paste that token into the setup page and create your administrator account.
From there, follow BookOrbit's own docs to create your first library:
https://bookorbit.app/creating-a-library

## Notes / things worth adjusting

- **Image tag**: this pins `ghcr.io/bookorbit/bookorbit:latest`. For a more
  predictable install, change it to a specific release tag (check
  https://github.com/bookorbit/bookorbit/releases) before installing —
  updates then happen by bumping the tag and reinstalling/updating rather
  than by whatever `latest` happens to point to.
- **Icon/gallery**: the manifest references `1.jpg`/`2.jpg`/`3.jpg` and no
  icon — Umbrel will just show a placeholder tile until you drop a
  `icon.svg` (256x256) and gallery screenshots (1440x900 jpgs) into the
  `custom-bookorbit/` folder. Purely cosmetic, safe to skip.
- **External database**: if you'd rather point at a Postgres you already
  run instead of the bundled container, drop the `postgres:` service and
  set `DATABASE_URL` on `web` instead — see the "External Database" section
  of BookOrbit's install docs.
- **Books folder**: books live at `~/umbrel/app-data/custom-bookorbit/data/books`
  on the Umbrel itself. If your books already live elsewhere on the box
  (e.g. a network share mounted into Umbrel), point the `volumes:` entry
  for `/books` at that path instead of the default.
