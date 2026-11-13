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

1. ~~apps nginx `:8080` transition publish~~ — DONE (2026-11-13): nginx s6
   service removed from the baked image; `:8080` publish removed from the
   devtools compose. codery-apps:20261112 ships neither.
2. remi_answer_engine_* volumes (3) — pending Rob's inspect-then-delete
   verdict (defunct past project).

## P4 completion notes (2026-11-11/12)

- REMI helper UIs (qdrant/ideas/booksearch/graph/bolt) live INSIDE the
  remi_backend compose project — one project, one file
  (`projects/REMI/docker-compose.yml`, committed to the REMI repo), 14
  containers, all visible in the single Dockhand `remi_backend` entry
  (sourceType: internal — imported in place, file stays in the repo).
- **Deploy = Dockhand Deploy button** for everything REMI, backend rebuilds
  included. Dockhand v1.0.57 deploys imported stacks with the ORIGINAL
  compose location as working directory — relative binds verified via
  container inspect (`/opt/codery/projects/REMI/...`, not /app/data).
  Dockhand older than ~1.0.13 would misresolve them to /app/data/stacks.
- Zoraxy upstreams for the helpers are **service-name aliases**
  (qdrant-proxy:8125, ideas:8124, booksearch:8126, graph-proxy:8127,
  bolt-proxy:8128) — stable across recreates; NO container_name anywhere in
  the file (fixed names were a cross-project collision foot-gun, removed).
- frontend service starts with `astro dev --force`: the persistent
  frontend_node_modules volume keeps Astro's dev-server lock across
  recreates, and a stale lock otherwise crash-loops the container.
- booksearch container: python:3.12-slim, boot-time `pip install sqlite-web`
  with HOME=/tmp (non-root pip), CLI at /tmp/.local/bin/sqlite_web. The
  booksearch-venv in the projects tree is no longer used (harmless leftover).
- The interim "remi" Dockhand stack and its 7 empty remi_* volumes were
  removed (2026-11-12). The 3 remi_answer_engine_* volumes are from a defunct
  past project — pending Rob's inspect-then-delete verdict.
- Dockhand API notes: image pull = POST /api/images/pull?env=1
  {"image","tag"}; stack create = POST /api/stacks {"name","compose"};
  stack compose update = PUT /api/stacks/<name>/compose {"content"};
  container delete = DELETE /api/containers/<id>; Zoraxy proxy delete =
  POST /api/proxy/del {"ep"}.

## Nix store drift incident (2026-11-12/13)

The 20261112 apps image deploy skipped the `/nix` store sync (the mandatory
step had been dropped in the RUNBOOK rewrite) — root process failure. Chain
of fallout, all fixed:

- **Shell/PATH**: gem's home dotfiles symlink into the new image's
  home-manager generation (`80g6dilvk-...`) → absent from the old volume
  store → dangling → bare sshd PATH → mosh (`NoMoshServerArgs`),
  `tooladd`, coreutils all unreachable. Recovered by force-copying the
  missing generation into the volume (OpenMINIS run; an earlier
  docker-run copy silently failed to persist — cause never determined —
  which is why every recovery step now verifies at the raw-volume AND
  exec layers).
- **nix config**: the nixos/nix base ships `/etc/nix/nix.conf` as a symlink
  INTO the store (`<hash>-base-system/...`) → post-drift it dangled → every
  bare `nix` call died with "experimental Nix feature 'nix-command' is
  disabled". Fixed live: real file written at `/etc/nix/nix.conf`
  (`experimental-features = nix-command flakes`).
- **tooladd**: gem-level `nix profile list` (manifest step) cannot take
  root's big-lock → noisy failure AFTER a successful install; the
  "already installed" check silently failed the same way. Repo fix: ALL
  nix store-db ops run under sudo; manifest write + git stay as gem.
- `vim` added to the tools profile via tooladd (manifest now cloc + vim).

**Repo fixes committed** (ride the next image build; live stopgap already
in place): Dockerfile bakes a real `/etc/nix/nix.conf` (rm -f first — the
base's store symlink); tooladd/tools-replay pass explicit
`--extra-experimental-features` flags and use `add` not `install`;
tools-replay had a POSIX violation (`<<<` in a /bin/sh script) fixed;
`nix-store-sync.sh` now does a FULL `cp -a` (no `-n` skip) with in-run
verification; RUNBOOK reinstates the mandatory sync step with the failure
class documented.

**Ops rule**: every apps image deploy runs
`host/scripts/nix-store-sync.sh <new-image> codery-apps-nix` BEFORE
deploying, then verifies `ssh gem@apps 'echo $PATH'` includes the nix
profile dirs.
