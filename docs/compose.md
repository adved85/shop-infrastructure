# `compose.yml` and `compose.override.yml`

Related: [nginx.md](./nginx.md) · [environment.md](./environment.md) · [roadmap.md](./roadmap.md) · [hands-on.md](./hands-on.md) · [README](../README.md)

## Why two files instead of one

`compose.yml` is written to be **safe to run on a real, internet-facing server**. On its own,
it only publishes ports `80` and `443` (the `proxy` service) — every other service
(Postgres, Redis, RabbitMQ) is reachable *only* from other containers on the internal
Docker network, never from the host machine or the internet.

`compose.override.yml` adds back host-port mappings for Postgres, Redis and RabbitMQ,
purely for local convenience (so you can point a GUI tool like TablePlus, RedisInsight,
etc. at `localhost`). Docker Compose automatically merges an override file that sits next
to `compose.yml` in the same directory — you don't need to pass any extra flag locally:

```bash
# local (compose.yml + compose.override.yml merged automatically)
docker compose --env-file .env.prod up -d

# production (compose.override.yml explicitly excluded)
docker compose -f compose.yml --env-file .env.prod up -d
```

The reasoning behind this split is explained in more detail in
[environment.md](./environment.md#local-vs-production-ports).

## Services

| Service    | Image                                       | Role |
|------------|----------------------------------------------|------|
| `proxy`    | `nginx:1.31-alpine`                           | Single public entry point. Routes `/` to `frontend`, `/api/` to `api`. See [nginx.md](./nginx.md). |
| `frontend` | `ghcr.io/adved85/react-shop-client`           | Static React/Vite build, served by its own bundled nginx (port 80 internally). |
| `api`      | `ghcr.io/adved85/laravel-shop-api`            | Laravel application, running php-fpm (port 9000, FastCGI — not HTTP). |
| `postgres` | `postgres:18`                                 | Primary datastore for the Laravel app. |
| `redis`    | `redis:8-alpine`                              | Cache. |
| `rabbitmq` | `rabbitmq:4-management`                       | Queue/messaging broker, for async work between Laravel and future Go/Rust workers. The `-management` variant enables the `rabbitmq_management` plugin (web UI/HTTP API on `15672`) on top of the `rabbitmq_prometheus` plugin the plain image already ships with. |

### Why the image tags are pinned

None of these use floating tags (`latest`, `alpine`) on purpose. A floating tag means a
`docker compose pull` can silently move you across a major version — which for **Postgres**
is genuinely dangerous: it refuses to start against a data directory initialized by an older
major version, so the database simply won't come up until you perform a dump/restore
migration. Pinning the major line still lets patch and minor updates through (so security
fixes keep arriving), while ruling out surprise major jumps.

Two deliberate choices worth noting:

- **`postgres:18`, not `postgres:18-alpine`.** The Alpine variant uses musl instead of glibc,
  which changes string collation behaviour (and therefore index ordering). The Debian-based
  image is the safer default for a database.
- **`nginx:1.31-alpine`, not `nginx:1-alpine`.** nginx has been on major version 1 for its
  entire life, so pinning the major gives no protection at all — the minor line is the
  meaningful unit of stability here.

`api` and `frontend` are never *built* here — this repo only ever pulls pre-built,
pre-tested images published by each app repo's own CI (see
[environment.md](./environment.md#api_version--frontend_version)). Infrastructure and
application code are deliberately kept in separate repositories:

- API source: https://github.com/adved85/laravel-shop-api
- Frontend source: https://github.com/adved85/react-shop-client

## Startup order: healthchecks, not just start order

Every service has a real `healthcheck:`, and every `depends_on` uses
`condition: service_healthy` rather than plain start-order. Concretely:

```
postgres, redis, rabbitmq  (healthy) ──▶ api  (healthy) ──▶ proxy
                                          frontend (healthy) ──▶ proxy
```

This means `api` will not even attempt to start until Postgres, Redis and RabbitMQ all
report healthy, and `proxy` won't start until *both* `api` and `frontend` report healthy —
so nginx never comes up routing traffic to a backend that isn't actually ready yet.

### What actually makes a check pass: the exit code

Docker doesn't read the output of a healthcheck command. It reads the **process exit
code**, and nothing else:

| Exit code | Docker's interpretation |
|---|---|
| `0` | healthy |
| `1` | unhealthy |
| `2` | reserved — don't use it |

Everything else is built on that one rule, which is why each check below is shaped the way
it is:

- `redis-cli ping` and `pg_isready` already exit non-zero when they can't reach their
  server, so they work as healthchecks unmodified.
- `wget --spider` (and `curl -f`) are used for the HTTP checks specifically **because they
  exit non-zero on an HTTP error status**. A plain `curl` would happily fetch a `503` page
  and exit `0`, reporting a broken service as healthy. This is also what makes the API's
  `/health/ready` returning `503` meaningful rather than cosmetic.
- `php artisan health:check` ends with:

  ```php
  return $result['status'] === 'ok' ? self::SUCCESS : self::FAILURE;
  ```

  Those constants are Symfony Console's `SUCCESS = 0` and `FAILURE = 1` — chosen precisely
  to line up with what Docker expects. The human-readable `database: ok` / `rabbitmq: error`
  lines the command prints are for *you*, when you run it by hand or read
  `docker inspect`'s health log; Docker itself only cares that the process exited `0` or `1`.

Two related details on timing: a check has to fail `retries` times **consecutively** before
the container flips to unhealthy, and failures during `start_period` don't count toward that
at all — so a slow-booting service isn't punished for not being ready in its first seconds.

What each healthcheck actually does, and why the command differs per service:

- **`postgres`**: `pg_isready -U $POSTGRES_USER -d $POSTGRES_DB` — checked as a shell
  variable expanded *inside* the container (from its own `environment:`), not a Compose
  `${VAR}`, so it always matches whatever the container is actually running with.
- **`redis`**: `redis-cli ping`.
- **`rabbitmq`**: `rabbitmq-diagnostics -q ping`.
- **`proxy`**: `wget --spider http://localhost/nginx-health` — a dedicated endpoint (see
  [nginx.md](./nginx.md)) that checks nginx itself, independent of whether
  `frontend`/`api` are healthy.
- **`frontend`**: `wget --spider http://localhost/` — its own bundled nginx serves the
  static build directly, so a plain HTTP check is meaningful here.
- **`api`**: `php artisan health:check` — see the story below.

### How the `api` healthcheck got better

This one is worth reading as a sequence, because the first version was weak for a reason
and it took a change in the *application* repo to fix it.

**Where it started: `nc -z 127.0.0.1 9000`.** The `api` container runs php-fpm and nothing
else. php-fpm speaks FastCGI, a binary protocol — so `curl` and `wget` cannot talk to it at
all, no matter what URL you give them, and the image contains no FastCGI client. That left
a bare TCP connect as the strongest check available: it proved the port was open and
accepting connections, and nothing more. A Laravel app that booted fine but couldn't reach
Postgres would still have reported perfectly healthy.

**What changed.** `laravel-shop-api` gained a `ReadinessChecker` service that verifies the
app can actually reach Postgres, Redis and RabbitMQ, exposed two ways: as HTTP routes
(`/health/live`, `/health/ready`) and as a console command (`php artisan health:check`).
The console form is the one that matters here — it runs entirely in-process, so it needs no
HTTP listener, no hostname and no port, which is exactly the constraint that blocked us
before. Full write-up lives in the `laravel-shop-api` repo:
[`app/docs/guide/health_checks.md`](https://github.com/adved85/laravel-shop-api/blob/main/app/docs/guide/health_checks.md).

**What we run now.** `php artisan health:check` — a genuine application-level probe. It
exits non-zero if any dependency is unreachable, which is what Docker reads as unhealthy.

Three consequences of that upgrade, all visible in `compose.yml`:

1. **`api` now `depends_on` RabbitMQ too.** Its healthcheck is a *readiness* check, so it
   cannot pass until all three backing services are reachable. Without RabbitMQ in the
   dependency list, a cold `up` could start `api` too early; it would fail readiness, never
   report healthy, and `proxy` — which gates on `api: service_healthy` — would then never
   start at all.
2. **The timeouts are layered deliberately.** `ReadinessChecker` gives each dependency 3s,
   so Docker's `timeout:` is set to 10s — comfortably above it. Set them the other way
   around and Docker kills the probe *before* the app can finish diagnosing itself, turning
   an informative "rabbitmq: error" log line into a silent timeout. The same rule governs
   `fastcgi_read_timeout` in [nginx.md](./nginx.md).
3. **`interval` went from 10s to 30s.** Each probe boots the Laravel framework, which is far
   heavier than the old TCP connect. Readiness doesn't need ten-second granularity.

**Liveness vs. readiness** — the app exposes both, and the difference is about what an
orchestrator should *do*:

| | Question | Correct reaction | Touches other services? |
|---|---|---|---|
| Liveness (`/health/live`) | Is the process alive? | Restart the container | **No** — deliberately |
| Readiness (`health:check`, `/health/ready`) | Can it serve traffic? | Stop sending traffic, don't restart | Yes — all three |

Liveness touching an external service is a classic trap: a database blip would mark every
healthy API container as dead and restart them all in a loop, turning a recoverable outage
into a self-inflicted one. Compose uses the readiness form here because its only consumer
is `depends_on: service_healthy`, which is genuinely asking "can this serve traffic yet?".

Note that Compose only ever uses the **console** form. The `/health/live` and
`/health/ready` HTTP endpoints are routed by the proxy and reachable by hand, but no
healthcheck in this file calls them — a Docker healthcheck runs inside the container it
checks, and `api` has no HTTP listener for one to talk to. See
[nginx.md](./nginx.md#who-calls-these-nothing-yet--and-thats-deliberate) for who those
endpoints are actually for.

## Volumes

Four named volumes persist data across container restarts: `redis-data`, `postgres_data`,
`rabbitmq-lib`, `rabbitmq-log`. They live in Docker's local storage and are not something
you need to manage manually.
