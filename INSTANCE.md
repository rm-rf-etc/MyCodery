# INSTANCE.md — this deployment's inventory

The running Codery instance this repo describes. Product:
[CoderyOSS/Codery](https://github.com/CoderyOSS/Codery) — cloned separately at
`/workspace/Codery` (public `main`); this repo is standalone (no fork, no
shared history) and holds only instance facts.

## Identity

| | |
|---|---|
| Domain | `rancidgrandmas.online` (wildcard TLS, Cloudflare DNS-01) |
| Host | netcup VPS, 16 GiB RAM · tailnet IP in `/opt/codery/.env` (`TAILNET_IP`) |
| Operator | Rob · agents spawn via paseo (OpenMINIS from iPad) |

## Stacks

| Stack | Services | Notes |
|---|---|---|
| `dockhand` (core) | dockhand | host-side compose; UI `127.0.0.1:23000` + dockhand.<domain> |
| `zoraxy` (core) | zoraxy | tailnet `:443/:80`, mgmt `:8000` |
| `devtools` (tools) | paseo, apps + profiles `browser,design,db` (playwright, opendesign, dbgate) | `COMPOSE_PROFILES=browser,design,db` |
| `remi_backend` (project) | qdrant, mongo, memgraph, memgraph-lab, api, worker, ingest_worker, frontend, backup | compose at REMI repo root |
| `carta` (project) | carta-client, carta-design, carta-cbe1 | compose at Carta repo root |

## Vhosts (Zoraxy)

code. → paseo:6767 · dbgate. → dbgate:3000 · dockhand. → dockhand:3000 ·
remi. → remi_backend-frontend-1:4321 · opendesign. → opendesign:7456 ·
qdrant. → remi-qdrant-proxy:8125 · ideas. → remi-ideas:8124 ·
booksearch. → remi-booksearch:8126 · graph. → remi-graph-proxy:8127 ·
bolt. → remi-bolt-proxy:8128 · routing. → zoraxy:8000

(`label.` retired with the labeler app — Zoraxy host removed 2026-11-10.
Zoraxy proxy-delete API: `POST /api/proxy/del` with form param `ep=<domain>`.)

## .env keys beyond templates

`ACME_EMAIL` (zoraxy bootstrap), `ZORAXY_ADMIN_USER` / `ZORAXY_ADMIN_PASSWORD`
(zoraxy bootstrap-state.sh), `TAILNET_IP` (stack .envs), `DOCKHAND_API_TOKEN`
(devtools stack .env → paseo MCP).

## P3 notes (2026-11-11)

- **paseo→apps SSH**: phase C created `/opt/codery/ssh/agent-to-apps/` as an
  EMPTY directory (keypair never staged) — ssh was silently broken since the
  migration. Fixed 2026-11-11: fresh keypair generated in the paseo home
  volume (`~/.ssh/agent-to-apps.key` + `~/.ssh/config`), pubkey appended to
  `/opt/codery/ssh/authorized_keys` on the host. Volume-resident config
  survives recreates; the compose mount of the empty dir is now vestigial.
- **apps SSH host keys**: changed once at the phase-C transition (Nov 9) and
  correctly persisted through today's recreate — one-time artifact, volume
  persistence confirmed working.

## Deployment decisions (2026-11-10, P2)

- **opendesign builds on-host (Option B)**: the devtools runtime compose
  (Dockhand copy) carries a `build:` section — context
  `/opt/codery/projects/Codery/images/opendesign`, image tag `codery-od:local`.
  No ghcr dependency for opendesign. Update path: edit the Dockerfile pin →
  Dockhand Deploy with *build* enabled.
- **dbgate**: pre-provisioned connection must be named WITHOUT dashes
  (`remi`, not `remi-mongo`) — dbgate's env-key lookup silently yields an
  undefined engine for dashed names. Connection lives in the runtime compose
  (instance config); REMI Mongo is authless on the internal network.
- **Dockhand deploys are local-first**: the dockhand MCP `stack_sync` defaults
  to `pull:false` — private ghcr images (paseo/apps) need either registry
  creds in Dockhand (Settings → Registries, GHCR_USERNAME/GHCR_TOKEN) or a
  host-side `docker pull` before deploying refreshed images.
- paseo/apps were not recreated in P2 (config-hash preserved); their ghcr
  images remain the pre-P2 versions.

## Known debt / pending work

1. apps nginx `:8080` transition publish — retire after final route audit
   (no vhost targets it since P4)

## P4 notes (2026-11-11)

- REMI helper UIs (qdrant/ideas/booksearch/graph/bolt) migrated out of the
  apps container into this stack as bind-mounted containers
  (remi-qdrant-proxy / remi-ideas / remi-booksearch / remi-graph-proxy /
  remi-bolt-proxy); Zoraxy upstreams re-pointed; apps ports 8124-8128
  retired (s6 bundles removed, ports refused).
- **Dockhand "remi" stack**: remi_backend was ADOPTED into Dockhand without
  a compose location, so it cannot deploy. A sibling stack "remi" was
  registered from an ABSOLUTIZED copy of the repo compose (all binds, build
  contexts and env_files → /opt/codery/projects/REMI/...); deploying it
  reconciles the same compose project (`name: remi_backend` in the file)
  and is the ONLY deploy path for REMI until Dockhand supports
  adopted-stack locations. The old "remi_backend" record is a cosmetic
  duplicate view of the same containers.
- REMI backend rebuilds work from the "remi" stack (absolute build
  contexts in the stored copy).
- booksearch container: python:3.12-slim, boot-time `pip install sqlite-web`
  with HOME=/tmp, CLI at /tmp/.local/bin/sqlite_web. The booksearch-venv in
  the projects tree is no longer used by the service (harmless leftover).
- Empty leftover volumes from an aborted duplicate project (remi_qdrant_data,
  remi_model_cache, …) are harmless; host-side `docker volume rm` optional.
- Dockhand API notes: image pull = POST /api/images/pull?env=1
  {"image","tag"}; stack create = POST /api/stacks {"name","compose"};
  stack compose update = PUT /api/stacks/<name>/compose {"content"};
  container delete = DELETE /api/containers/<id>; Zoraxy proxy delete =
  POST /api/proxy/del {"ep"}.
