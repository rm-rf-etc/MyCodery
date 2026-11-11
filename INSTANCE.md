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
qdrant. → apps:8125 · ideas. → apps:8124 · booksearch. → apps:8126 ·
graph. → apps:8127 · bolt. → apps:8128 · routing. → zoraxy:8000

(`label.` retired with the labeler app — Zoraxy host removed 2026-11-10.
Zoraxy proxy-delete API: `POST /api/proxy/del` with form param `ep=<domain>`.)

## .env keys beyond templates

`ACME_EMAIL` (zoraxy bootstrap), `ZORAXY_ADMIN_USER` / `ZORAXY_ADMIN_PASSWORD`
(zoraxy bootstrap-state.sh), `TAILNET_IP` (stack .envs), `DOCKHAND_API_TOKEN`
(devtools stack .env → paseo MCP).

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

1. **REMI helpers in apps** (qdrant/graph/bolt proxies, ideas, booksearch on
   8123-8128) — layer violation; migrate into `remi_backend`, re-point Zoraxy
2. paseo/apps visibility mounts + relocated opencode-config path + profiles
   (COMPOSE_PROFILES) — staged, land with the next gated devtools deploy
5. Host-side `rm -rf` needed for root-owned leftovers: `/opt/codery/projects/
   Codery/system/` (3.7 GB build remnants) + `/opt/codery/projects/Codery.old/`
   (64 MB pre-split tree; paseo-owned parts removed, root-owned node_modules +
   skills/ remain — the live paseo config mount pins this inode, so leave until
   the next paseo recreate, then remove both)
6. apps nginx `:8080` transition publish — retire after route audit
