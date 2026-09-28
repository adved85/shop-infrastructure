# Changelog

All notable changes to this project are documented in this file, newest
first. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

The reasoning behind each piece lives in [docs/](docs/); what's still ahead
is in [docs/roadmap.md](docs/roadmap.md).

## [Unreleased]

## [0.1.0] - 2026-09-28

The first release, built in two steps.

### Step 2 — migrations, logs and runbooks - 2026-09-26

The stack now starts from an empty database to a working login.

#### Added

- One-shot `migrate` service that runs migrations on every `up`. It shares
  `api`'s database settings through a YAML anchor (`x-db-env`), and `api`
  starts only if it succeeds.
- Laravel logs go to stderr, so `docker compose logs api` shows them, with
  Docker log rotation at 10 MB × 3 files.
- Restart policies for `api` and `frontend`.
- Docs: a first-start runbook from an empty database to a working login;
  database export, update and recreate; `/api/` documented as a contract
  between the frontend, the proxy and Laravel.

#### Changed

- `compose.override.yml` became `compose.dev.yml`. Local extras are now
  opt-in (`-f compose.yml -f compose.dev.yml`, or `COMPOSE_FILE`), so a
  plain `docker compose up` is production-safe.
- `.env.prod` became `.env`, which Compose loads automatically, so
  `--env-file` is no longer needed.
- Debug mode (`APP_ENV=local`, `APP_DEBUG=true`) is on only in
  `compose.dev.yml`.

#### Fixed

- `api` now receives `APP_KEY`. Without it the container stayed healthy
  while every request that needed encryption failed.
- Healthchecks probe `127.0.0.1` instead of `localhost`. Inside Alpine
  containers `localhost` resolves to IPv6 `::1`, but nginx listens only on
  IPv4.

### Step 1 — Compose stack, nginx proxy and infrastructure docs - 2026-09-23

Six services wired together with Docker Compose: an nginx proxy, the React
frontend and Laravel API pulled as published GHCR images, plus Postgres,
Redis and RabbitMQ.

#### Added

- `compose.yml`, production-safe on its own: only the proxy publishes
  ports. `compose.override.yml` published Postgres, Redis and RabbitMQ on
  localhost for local work and was excluded in production with
  `-f compose.yml`.
- Per-service environment variables instead of a blanket `env_file`, so no
  container holds a secret it doesn't use. RabbitMQ's app-side credentials
  are mapped from the same values the broker uses, so the two can't drift
  apart. Laravel uses Redis for cache and RabbitMQ for queues.
- Health-gated startup order: `api` waits for Postgres, Redis and RabbitMQ,
  and `proxy` waits for `api` and `frontend`. The `api` probe is
  `php artisan health:check`, an application-level readiness check rather
  than a TCP ping.
- nginx routing: `/` to the frontend over HTTP, `/api/` to php-fpm over
  FastCGI. The `/health/live` and `/health/ready` probe routes are reachable
  only from inside the container.
- Infrastructure images pinned to explicit versions. With floating tags, a
  pull could move Postgres to a major version it can't start against.
- `.env.example` documenting every variable; `.env.prod` held the real
  secrets and was git-ignored.
- `docs/` explaining the reasoning behind each piece, including the
  non-obvious parts and the ones that caused real bugs along the way.

[Unreleased]: https://github.com/adved85/shop-infrastructure/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/adved85/shop-infrastructure/releases/tag/v0.1.0
