# Changelog

What each step added to the stack, newest first. The reasoning behind each
piece lives in [docs/](docs/); what's still ahead is in
[docs/roadmap.md](docs/roadmap.md).

## Step 2 — migrations, logs and runbooks (2026-09-26)

The stack now starts from an empty database to a working login.

- **Migrations:** a one-shot `migrate` service runs migrations on every `up`.
  It shares `api`'s database settings through a YAML anchor (`x-db-env`),
  and `api` starts only if it succeeds.
- **Logs:** Laravel logs go to stderr, so `docker compose logs api` shows
  them, with Docker log rotation at 10 MB × 3 files. Debug mode is on only
  in the local dev file.
- **Dev file:** `compose.override.yml` became `compose.dev.yml`. Local extras
  are now opt-in (`-f compose.yml -f compose.dev.yml`, or `COMPOSE_FILE`),
  so a plain `docker compose up` is production-safe.
- **Env file:** `.env.prod` became `.env`, which Compose loads automatically,
  so `--env-file` is no longer needed.
- **`APP_KEY`:** now passed to `api`. Without it the container stayed
  healthy while every request that needed encryption failed.
- **Healthchecks** probe `127.0.0.1` instead of `localhost`. Inside Alpine
  containers `localhost` resolves to IPv6 `::1`, but nginx listens only on
  IPv4.
- **Restart policies** added for `api` and `frontend`.
- **Docs:**
  - a first-start runbook, from an empty database to a working login
  - database export, update and recreate
  - `/api/` documented as a contract between the frontend, the proxy and
    Laravel

## Step 1 — Compose stack, nginx proxy and infrastructure docs (2026-09-23)

Six services wired together with Docker Compose: an nginx proxy, the React
frontend and Laravel API pulled as published GHCR images, plus Postgres,
Redis and RabbitMQ.

- `compose.yml` is production-safe on its own: only the proxy publishes
  ports. `compose.override.yml` published Postgres, Redis and RabbitMQ on
  localhost for local work and was excluded in production with
  `-f compose.yml`.
- Each service gets only the environment variables it needs, instead of a
  blanket `env_file`. RabbitMQ's app-side credentials are mapped from the
  same values the broker uses, so the two can't drift apart. Laravel is told
  to use Redis for cache and RabbitMQ for queues.
- Startup order is gated on real health, not just container start: `api`
  waits for Postgres, Redis and RabbitMQ, and `proxy` waits for `api` and
  `frontend`. The `api` probe is `php artisan health:check`, an
  application-level readiness check rather than a TCP ping.
- nginx routes `/` to the frontend over HTTP and `/api/` to php-fpm over
  FastCGI. The `/health/live` and `/health/ready` probe routes are reachable
  only from inside the container.
- Infrastructure images are pinned to explicit versions. With floating tags,
  a pull could move Postgres to a major version it can't start against.
- `.env.example` documents every variable. `.env.prod` held the real secrets
  and was git-ignored.
- `docs/` explains the reasoning behind each piece, including the
  non-obvious parts and the ones that caused real bugs along the way.
