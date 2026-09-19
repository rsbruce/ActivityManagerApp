# Activity Manager

A local-first personal activity manager: a Vue 3 + Capacitor client backed by a
custom SQLite sync engine, so the same data works offline on device and syncs
across devices through a small self-hosted server.

This umbrella repo bundles the three components as git submodules and wires them
together with Docker Compose, so you can clone it and run the whole system with
one command.

## Architecture

Three independently-published repos, layered:

| Component | Repo | Role |
|---|---|---|
| **Sync engine** | [sync-engine-ts](https://github.com/rsbruce/sync-engine-ts) | The sync algorithm as a standalone library (`single-player-sync` on npm). Last-writer-wins over row-level `updated_at`; SQLite-agnostic via an adapter interface. Consumed by both the server and the client. |
| **Sync server** | [sync_server](https://github.com/rsbruce/sync_server) | A small Hono service (Node's built-in `node:sqlite`) holding one SQLite database per user. Symmetric `/sync` endpoint; JWT auth with rotating refresh tokens. |
| **Client** | [vue-activity-manager](https://github.com/rsbruce/vue-activity-manager) | Vue 3 (`<script setup>`) + Capacitor app. Runs against an in-browser SQLite (WASM) on the web and native SQLite on Android. |

**How they connect:** the client owns the source of truth locally and works fully
offline. When online and logged in, it pushes/pulls row deltas to the server via
the engine; the server applies them last-writer-wins and returns the other
device's changes. The engine is the shared brain that both sides run.

## Quick start

Requires Docker (with Compose v2).

```bash
git clone --recurse-submodules https://github.com/rsbruce/activity-manager.git
cd activity-manager
docker compose up
```

Then open **http://localhost:5173**.

> Already cloned without `--recurse-submodules`? Run
> `git submodule update --init --recursive`.

Two processes come up:

- **web** — the Vue app (Vite dev server) on `http://localhost:5173`
- **server** — the sync backend on `http://localhost:9000`

The app is **local-first**, so it's fully usable the moment it loads — no account
needed. Everything you create is stored in the browser.

## Enabling sync (optional)

Sync is opt-in. To turn it on:

1. In the app, open **Sync settings**.
2. Register / log in with any username + password, plus the **signup secret**
   (the dev default is `letmein`, set by `SIGNUP_SECRET`).

That creates your account and a server-side database on the fly, and the app
starts syncing. Open the app in a second browser (or a private window) and log in
with the same account to watch changes propagate.

## Configuration

`docker compose up` runs with insecure **dev defaults** baked into
`docker-compose.yml`, so it works with zero setup. To override, copy
`.env.example` → `.env` and edit. Key values:

| Variable | Default | Meaning |
|---|---|---|
| `JWT_SECRET` | `dev-only-insecure-change-me` | Signs access tokens. **Change for any real use.** |
| `SIGNUP_SECRET` | `letmein` | Gate for account creation. |
| `ALLOWED_ORIGINS` | `http://localhost:5173` | CORS origins allowed to call the server. |
| `VITE_SYNC_URL` | `http://localhost:9000` | Backend URL the client talks to. |

## Notes

- **This is a dev/review setup.** The web app runs via the Vite dev server
  (unminified, hot-reload) — deliberately transparent and quick to start, not a
  production build. The default secrets are insecure by design.
- **Submodules pin exact commits.** To move a component to its latest, `cd` into
  the submodule, `git pull`, then commit the updated pointer here.
- The Android build of the client is out of scope for this stack; see the client
  repo for Capacitor build instructions.

## License

MIT — see [LICENSE](LICENSE). Each submodule is MIT-licensed in its own repo.
