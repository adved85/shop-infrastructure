# shop-infrastructure

## What this is

This repository is the **infrastructure layer** for an online shop application, deliberately
kept separate from the application code itself. It owns orchestration (Docker Compose),
the reverse proxy configuration, and environment/deployment concerns — nothing else. It
does not contain any application source code.

The problem this split solves: infrastructure and application code change for different
reasons, at different speeds, and shouldn't be coupled. This repo never builds an
application image itself — it only ever *pulls* a specific, already-tested, versioned
image published by each application repo's own CI. That means this repo can be deployed,
rolled back, or reconfigured without touching application source, and each application repo
can be developed, tested and released independently of how or where it's deployed.

The two application repositories this infrastructure serves:

- **API** — Laravel backend: https://github.com/adved85/laravel-shop-api
- **Frontend** — React/Vite client: https://github.com/adved85/react-shop-client

## Architecture

```
                        ┌──────────┐
        client ───────▶ │  proxy   │  (nginx, ports 80/443 — the only public entry point)
                        └────┬─────┘
                   ┌─────────┴─────────┐
                   ▼                   ▼
             /  → frontend       /api/ → api
          (React, own nginx)   (Laravel, php-fpm)
                                       │
                        ┌──────────────┼──────────────┐
                        ▼              ▼              ▼
                   postgres         redis         rabbitmq
                  (database)       (cache)   (queues / messaging,
                                              for future Go/Rust workers)
```

| Service    | Role |
|------------|------|
| `proxy`    | nginx. The only container reachable from outside — routes `/` to `frontend`, `/api/` to `api`. |
| `frontend` | React/Vite build, served by its own bundled nginx. |
| `api`      | Laravel application (php-fpm). |
| `postgres` | Primary datastore. |
| `redis`    | Cache. |
| `rabbitmq` | Queue/messaging broker. Now Laravel's configured queue driver (`QUEUE_CONNECTION=rabbitmq`), though no jobs are dispatched to it yet — the async work it's intended for (order events, Go/Rust workers) is still ahead. |

Full details on each piece, including the non-obvious parts (FastCGI vs. HTTP, why
`frontend` needs no runtime environment variables, healthcheck gating, why ports are split
across two files): see [docs/compose.md](docs/compose.md),
[docs/nginx.md](docs/nginx.md), and [docs/environment.md](docs/environment.md).
Where this is headed next: [docs/roadmap.md](docs/roadmap.md).

## Files in this repo

| File | Purpose |
|---|---|
| `compose.yml` | The production-safe base. Only publishes `80`/`443`. → [docs/compose.md](docs/compose.md) |
| `compose.dev.yml` | Local development only: publishes `5432`/`6379`/`5672`/`15672` to `localhost` and turns on the API's debug mode. Applied only when passed explicitly with `-f`, **never in production**. |
| `proxy/default.conf` | nginx routing config. → [docs/nginx.md](docs/nginx.md) |
| `.env.example` | Template for the real env file — committed, no real secrets. → [docs/environment.md](docs/environment.md) |
| `.env` | Real secrets. **Git-ignored, never commit this.** |
| `docs/hands-on.md` | Commands to validate config before `up`, and to inspect/debug the running stack. |
| `docs/roadmap.md` | Where this project is headed — done/next/future phases. |
| `docs/plans/` | Raw working notes behind the roadmap (git-ignored — personal reference, not published). |

## Local vs. production — what actually differs

The same two application images run in both places, unchanged. Only this differs:

| | Local | Production |
|---|---|---|
| Command | `docker compose -f compose.yml -f compose.dev.yml up -d` | `docker compose up -d` |
| Postgres / Redis / RabbitMQ ports | published to `localhost` via `compose.dev.yml` | not published at all |
| Reached at | `http://localhost` | `https://yourdomain.com` |
| TLS | none | domain + certificate (not yet set up) |
| `.env` contents | throwaway secrets | real secrets |

Note which one needs the extra typing. `compose.dev.yml` is deliberately *not* named
`compose.override.yml` — Compose would auto-merge that one with no flags, which would make
the convenient command and the dangerous command the same command. As it stands, forgetting
a flag on the server changes nothing; forgetting one locally just means a GUI tool can't
reach Postgres.

This only works because the frontend calls the API at the **relative** path `/api` rather
than a hardcoded hostname. The browser resolves that against whichever origin served the
page — `http://localhost/api/...` locally, `https://yourdomain.com/api/...` in production —
so one published image is correct everywhere, and there's no cross-origin request and
therefore no CORS configuration anywhere in the stack. That value is baked into the bundle
at build time, from the `ARG VITE_API_URL=/api` default in `react-shop-client`'s Dockerfile;
changing it requires publishing a new frontend release, not editing anything here.

## Getting started — local

```bash
git clone https://github.com/adved85/shop-infrastructure.git
cd shop-infrastructure
cp .env.example .env
# edit .env: replace every CHANGE_ME with a real value
#   openssl rand -base64 32

export COMPOSE_FILE=compose.yml:compose.dev.yml   # optional: saves repeating -f
docker compose config --quiet && echo "config OK"
docker compose up -d
docker compose ps          # wait for everything to report "healthy"
```

Without that `export`, pass the files explicitly each time:
`docker compose -f compose.yml -f compose.dev.yml up -d`

**Next: [First start](docs/hands-on.md#first-start--from-an-empty-database-to-a-working-login)**,
the steps from an empty database to a logged-in admin, in order. Migrations have already
run by this point.

If something doesn't come up healthy — or you just want to look around inside the running
stack — [docs/hands-on.md](docs/hands-on.md) has the commands for inspecting containers,
the network, and each service directly.

Then:
- App: http://localhost/
- API: http://localhost/api/...
- Postgres/Redis/RabbitMQ are also reachable on `localhost` at their default ports
  (courtesy of `compose.dev.yml`) — see
  [docs/environment.md](docs/environment.md#software-for-the-locally-exposed-ports) for
  suggested GUI tools (TablePlus/DBeaver for Postgres, RedisInsight for Redis).

To tear down: `docker compose down` (add `-v` to also delete the named volumes/data).

## Getting started — production

```bash
# on the server, after Docker + the Compose plugin are installed:
git clone https://github.com/adved85/shop-infrastructure.git
cd shop-infrastructure
cp .env.example .env   # or scp a prepared .env from elsewhere — never commit it
# edit .env with real production secrets

# the plain command uses compose.yml only — compose.dev.yml is never
# applied unless explicitly passed, so Postgres/Redis/RabbitMQ stay unexposed
docker compose up -d
docker compose ps
```

HTTPS is not yet configured — it requires a real registered domain (Let's Encrypt cannot
issue a certificate for an IP address or `localhost`) plus actual certificate files before
`proxy/default.conf` can add a `listen 443 ssl` block. See
[docs/nginx.md](docs/nginx.md#https-443) for why this can't just be added speculatively.

Manual `deploy.sh`/`rollback.sh`/`healthcheck.sh` scripts, and eventually a proper
deployment tool, are the next step once the above has been run by hand a few times — see
[docs/roadmap.md](docs/roadmap.md) for the full plan.

## Nuances worth knowing before touching this repo

A handful of things that weren't obvious while building this out, collected here so they
don't have to be rediscovered:

- **`api` speaks FastCGI, not HTTP.** php-fpm listens on port `9000`; nginx must use
  `fastcgi_pass`, never `proxy_pass`, to reach it. See [docs/nginx.md](docs/nginx.md).
- **A missing `APP_KEY` is invisible to every health check.** The container starts, php-fpm
  serves, and `health:check` passes — but the app throws on anything needing encryption.
  It's also the one secret you must *not* rotate. See
  [docs/environment.md](docs/environment.md#app_key--the-one-that-fails-silently).
- **`frontend` needs no runtime environment variables.** Its API URL is baked into the
  built JS bundle at image build time by its own CI, not read at container start.
- **Every service only receives the env vars it actually needs** — no blanket `env_file:`.
  This is deliberate: a shared env file would leak every secret into every container,
  including ones (like `frontend`) that have no legitimate use for them. See
  [docs/environment.md](docs/environment.md#why-each-service-only-sees-the-variables-it-needs).
- **`ports:` only controls host/internet reachability** — container-to-container traffic
  always goes over the internal Compose network by service name, regardless of whether a
  port is published. This is why `compose.dev.yml` exists as a separate, opt-in file rather
  than editing `compose.yml` directly.
- **RabbitMQ's management UI needs the `-management` image variant.** The plain `rabbitmq`
  image ships only the `rabbitmq_prometheus` plugin, so port `15672` would answer nothing —
  which is why the image is pinned to `rabbitmq:4-management`. Note that acknowledgements,
  retries, dead-letter queues and durability are all *core* broker features and need no
  plugin at all.
- **Infrastructure images are pinned to a major/minor line, never `latest`** — a floating
  tag lets a `docker compose pull` jump Postgres across a major version, which it cannot
  start against an older data directory. See [docs/compose.md](docs/compose.md#why-the-image-tags-are-pinned).
- **Image versions are not arbitrary.** `API_VERSION`/`FRONTEND_VERSION` must match a tag
  each app repo's own CI actually published to GHCR (triggered by pushing a git tag, e.g.
  `v0.1.0`) — see [docs/environment.md](docs/environment.md#api_version--frontend_version).
- **Migrations run automatically, before the app starts.** A one-shot `migrate` service
  applies them on every `up`, and `api` waits for it to exit successfully. A failed
  migration keeps the app down rather than letting it run against a broken schema. See
  [docs/compose.md](docs/compose.md#the-migrate-service).
- **Healthchecks gate startup order**, not just container start order — `api` won't start
  before Postgres/Redis/RabbitMQ are healthy, and `proxy` won't start before `api`/`frontend`
  are. `api`'s check is `php artisan health:check`, a real application-level readiness probe
  rather than a TCP ping — the story of that upgrade is in
  [docs/compose.md](docs/compose.md#how-the-api-healthcheck-got-better).
- **RabbitMQ has two sets of credential variable names** — the broker reads
  `RABBITMQ_DEFAULT_*`, Laravel reads `RABBITMQ_USER`/`RABBITMQ_PASSWORD`. Passing only the
  first set silently falls back to `guest@127.0.0.1` and fails. See
  [docs/environment.md](docs/environment.md#rabbitmq-credentials-two-sets-of-names-for-one-account).

## Roadmap

Done / next / future phases (CI/CD, manual deploy scripts, choosing a deployment tool,
RabbitMQ + Go/Rust workers, gRPC, monitoring): [docs/roadmap.md](docs/roadmap.md).
