# SillyTavern — Base44 Dev Environment

## What This Is
SillyTavern is a Node.js/Express LLM frontend (v1.19.0). It serves a single-page web UI and proxies requests to user-configured LLM APIs (OpenAI, Anthropic, etc.). No database, cache, or external credentials are needed to boot — API keys are configured at runtime through the UI.

## Running
```bash
docker compose -f docker-compose.base44.yml up -d --build
```
- Web UI on **host port 3000** (mapped to container port 8000).
- Health: `curl http://localhost:3000/` → 200.

## Architecture
- **Base image**: `node:22` (not the project's production Dockerfile, which bakes a prebuilt image).
- **Source**: bind-mounted at `/home/node/app`; `node --watch` provides live-reload on server file changes.
- **Dependencies**: `npm ci --omit=dev --ignore-scripts` runs at container startup.
- **Webpack**: frontend libraries are compiled at server startup (cached in `data/_webpack/`). First compile ~15s; subsequent restarts use cache.
- **Config**: `config/config.yaml` is the source of truth. At startup, a copy is made to `/tmp/config-dev.yaml` with `basicAuthMode: false` and `securityOverride: true` (required for the preview to work without HTTP basic auth, which iframes can't display). A symlink `config.yaml → /tmp/config-dev.yaml` is created so `npm run init` and the server both read it.
- **CLI flags**: `--listen` (bind 0.0.0.0), `--heartbeatInterval 30` (enables healthcheck), `--browserLaunchEnabled false` (no browser in container).

## Key Files
- `server.js` — entry point; parses CLI args, sets `globalThis.DATA_ROOT` / `globalThis.COMMAND_LINE_ARGS`, dynamically imports `src/server-main.js`.
- `src/server-main.js` — Express app setup, middleware, routes, webpack compilation, server listen.
- `src/command-line.js` — CLI arg parser; falls back to `config.yaml` values for any arg not provided.
- `config/config.yaml` — all runtime configuration (port, auth, security, extensions, etc.).
- `webpack.config.js` — bundles vendored frontend libs (jquery, etc.) into `data/_webpack/<version>/output/lib.js`.
- `src/healthcheck.js` — checks heartbeat file freshness; used by compose healthcheck.

## No External Secrets Required
The app boots without any API keys. Users configure LLM provider credentials through the SillyTavern UI at runtime (stored in `data/`).

## Making Changes
- **Server code** (`src/*.js`, `server.js`): `node --watch` auto-restarts on save. Use `reload_preview` if the preview doesn't refresh.
- **Frontend static files** (`public/`): served directly by Express; browser refresh picks up changes.
- **Config changes**: restart the container (`docker compose -f docker-compose.base44.yml restart`).
- **Dependency changes**: rebuild (`docker compose -f docker-compose.base44.yml up -d --build`).

## Verification
```bash
docker compose -f docker-compose.base44.yml ps          # should show "healthy"
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/  # should be 200
```
