# Roadmap

Related: [compose.md](./compose.md) · [nginx.md](./nginx.md) · [environment.md](./environment.md) · [hands-on.md](./hands-on.md) · [README](../README.md)

A public, trimmed-down version of the working notes in `docs/plans/` (git-ignored, personal
scratch notes — this file is the version meant for anyone else reading this repo).

## Done

- [x] Registry + tag strategy — GHCR, tagged by pushing a git tag (`v*.*.*`), which
      triggers each app repo's `docker-publish.yml`.
- [x] `laravel-shop-api` and `react-shop-client` each have their own `Dockerfile`.
- [x] CI (`ci.yml`) and release publishing (`docker-publish.yml`) as separate workflows in
      both app repos — a release cannot publish without CI passing first.
- [x] `compose.yml` — all six services, wired together with least-privilege environment
      variables (no blanket `env_file:`).
- [x] `compose.override.yml` — local-only port exposure for Postgres/Redis/RabbitMQ,
      kept out of production via `docker compose -f compose.yml ...`.
- [x] `proxy/default.conf` — routes `/` to `frontend`, `/api/` to `api` via FastCGI.
- [x] Container/service-level healthchecks on all six services, with `depends_on:
      condition: service_healthy` gating startup order end-to-end.
- [x] `.env.example` + `docs/` written up.

## Next

- [ ] Provision a real server manually (install Docker + Compose plugin, open only
      `80`/`443` at the firewall, create a deploy user).
- [ ] First manual deploy on that server — copy `compose.yml` + a real `.env.prod`,
      `docker compose -f compose.yml --env-file .env.prod up -d`, verify by hand.
- [ ] Write `healthcheck.sh` (container health via `docker compose ps` / `docker inspect`,
      **plus** an end-to-end probe through the proxy against `/health/ready` — currently
      those HTTP endpoints exist but nothing calls them; this script is their first real
      consumer), `deploy.sh` (pull target version, `up -d`, run healthcheck, roll back on
      failure), `rollback.sh` (reset `API_VERSION`/`FRONTEND_VERSION` to the previous
      known-good tag, `up -d`).
- [ ] Repeat that deploy → healthcheck → rollback cycle by hand until it's second nature.
- [ ] Add a real `listen 443 ssl` server block, once a domain + certificate (Let's
      Encrypt/Certbot) exist — see [nginx.md](./nginx.md#https-443) for why this can't be
      added before then.

## Application-level health checks — done

The second of the three health layers is now built, in `laravel-shop-api`:

- **`php artisan health:check`** — a console readiness probe that runs in-process, needing
  no HTTP listener. This is what `api`'s Docker healthcheck runs, replacing the old
  `nc -z 127.0.0.1 9000` TCP check that could only prove the port was open.
- **`/health/live`** — liveness. Touches nothing external, so an outage elsewhere can't
  restart-loop a healthy container.
- **`/health/ready`** — readiness. Verifies Postgres, Redis and RabbitMQ are all reachable,
  returning `503` if any fail:
  ```json
  { "status": "ok", "checks": { "database": "ok", "redis": "ok", "rabbitmq": "ok" } }
  ```

All three share one `ReadinessChecker` service, so "ready" is defined in exactly one place.
Full write-up:
[`app/docs/guide/health_checks.md`](https://github.com/adved85/laravel-shop-api/blob/main/app/docs/guide/health_checks.md).
The infrastructure side — healthcheck wiring, timeout layering, the nginx probe routes — is
covered in [compose.md](./compose.md#how-the-api-healthcheck-got-better) and
[nginx.md](./nginx.md).

One thing still open:

- On the infrastructure side, only the **console** form is actually wired up — `api`'s
  Docker healthcheck runs `php artisan health:check`. The HTTP endpoints are routed and
  reachable (see [nginx.md](./nginx.md#who-calls-these-nothing-yet--and-thats-deliberate))
  but have no automated caller until `healthcheck.sh` exists.

## Later: evaluate a deployment tool

Once the manual `deploy.sh`/`rollback.sh` cycle has been run by hand enough times to
understand what it's actually automating, revisit tooling with that experience in hand.
Current thinking leans toward **Kamal** for the app-deploy layer (purpose-built for
zero-downtime Docker deploys — registry auth, rollback, health checks, proxy/SSL — with far
less config than Ansible), with **Ansible** reserved for the separate, much less frequent
job of provisioning the server itself (installing Docker, users, firewall rules). Not
decided yet — revisit once there's real deploy experience to judge it against.

## Later still: RabbitMQ, Go, Rust, gRPC

RabbitMQ is running and is now Laravel's configured queue driver
(`QUEUE_CONNECTION=rabbitmq`), but nothing dispatches jobs to it yet. The intended shape,
once there's a real use case to wire up (not before):

```
Laravel
   │
   ├─ immediate / synchronous work ──▶ gRPC ──▶ Go / Rust services
   │
   └─ async work / events ──▶ RabbitMQ ──┬─▶ Go   → warehouse processing
                                          ├─▶ Rust → analytics
                                          └─▶ PHP  → e.g. email
```

First real step here: wire one genuine RabbitMQ event in Laravel (e.g. `OrderCreated`) —
not an empty container — before adding any worker service. Go vs. Rust for a given piece of
work is a judgment call to make together when there's an actual task to assign, not a fixed
rule.

## Later still: infrastructure-level monitoring

Once the above is stable, the last layer is observability across the whole stack (not just
per-container healthchecks) — container resource metrics, dashboards, alerting. Candidate
toolset under consideration: **Prometheus + cAdvisor + Grafana**. Worth reassessing options
again once actually at this stage, rather than committing to a stack this far in advance.
