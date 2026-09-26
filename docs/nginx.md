# `proxy/default.conf`

Related: [compose.md](./compose.md) · [environment.md](./environment.md) · [roadmap.md](./roadmap.md) · [hands-on.md](./hands-on.md) · [README](../README.md)

`proxy` is the only service reachable from outside the Docker network (ports `80`/`443`).
Everything else — `api`, `frontend`, `postgres`, `redis`, `rabbitmq` — is only reachable
from other containers. This one nginx config is what decides where a request actually goes.

```nginx
location / {
    proxy_pass http://frontend:80;
}

location = /nginx-health {
    return 200 "ok\n";
    add_header Content-Type text/plain;
}

location = /health/live  { ...allow 127.0.0.1; deny all; fastcgi_pass api:9000; ... }
location = /health/ready { ...allow 127.0.0.1; deny all; fastcgi_pass api:9000; ... }

location /api/ {
    fastcgi_pass api:9000;
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME /var/www/html/public/index.php;
}
```

## `location /` → `frontend:80`

`react-shop-client`'s image bundles its **own** nginx, serving the pre-built Vite bundle,
listening on port `80` inside that container (confirmed from its Dockerfile: `EXPOSE 80`).
This is a plain HTTP reverse proxy — nothing unusual here.

The API base URL the frontend calls (`VITE_API_URL`) is baked into the built JavaScript
bundle at **image build time** in `react-shop-client`'s CI, not read from an environment
variable at container start. That's why `frontend` needs no runtime environment variables
at all (see [environment.md](./environment.md#why-each-service-only-sees-the-variables-it-needs)).

## `location = /nginx-health`

A small endpoint nginx answers directly, without proxying anywhere. It exists purely so
`proxy`'s Docker healthcheck (`docs/compose.md`) reflects **nginx's own** state, not the
state of whatever it happens to be routing to. If this checked `/` instead, `proxy` would
report "unhealthy" any time `frontend` had a problem, even though nginx itself was fine.

## `location = /health/live` and `= /health/ready`

Laravel exposes two probe endpoints (see
[`app/docs/guide/health_checks.md`](https://github.com/adved85/laravel-shop-api/blob/main/app/docs/guide/health_checks.md)
in the API repo). They live at the root, **not** under `/api/`, so without these blocks a
request to `/health/live` would fall through to `location /` and land on the *React app* —
which would cheerfully return its `index.html` and look like a passing check.

Two details that matter:

- **`allow 127.0.0.1; allow ::1; deny all;`** — the readiness response tells the caller
  which backing services are currently down, which isn't something to hand to the public
  internet. Restricting to loopback means only something running *inside* the proxy
  container can reach them: `docker compose exec proxy wget -qO- http://127.0.0.1/health/ready`
  for debugging, or a future healthcheck on the `proxy` service itself. External callers
  get a `403`. Relax the ACL to a monitoring system's address range if you later want
  external uptime checks.
- **`fastcgi_read_timeout 5s`** sits deliberately *above* `ReadinessChecker`'s own 3s
  per-dependency timeout. Set it lower and nginx severs the connection before Laravel can
  finish, so a slow dependency returns an opaque `504 Gateway Timeout` instead of the
  useful `503` with `{"rabbitmq": "error"}` in the body. Same layering principle as the
  Docker healthcheck timeout described in [compose.md](./compose.md#how-the-api-healthcheck-got-better).

### Who calls these? No Docker healthcheck — only you, for now

Worth stating plainly, because it's confusing otherwise: **no healthcheck in `compose.yml`
calls these two endpoints.** Every Docker healthcheck in the stack uses something else —

| Service | Runs | Uses `/health/*`? |
|---|---|---|
| `api` | `php artisan health:check` | No — console, in-process |
| `proxy` | `wget --spider .../nginx-health` | No — nginx's own endpoint |
| `frontend` | `wget --spider http://127.0.0.1/` | No |

There's a concrete reason for each. A Docker healthcheck runs *inside* the container it
checks, and the `api` container is php-fpm only — no HTTP server is listening in there, so
an HTTP probe against itself would connect to nothing. Hence the console command. And
`proxy` could reach Laravel over HTTP, but deliberately checks `/nginx-health` instead so
that nginx's health reflects nginx rather than its backends.

So today these routes exist for manual use, from inside the proxy container:

```bash
docker compose exec proxy wget -qO- http://127.0.0.1/health/ready
```

Their natural first automated consumer is the planned `healthcheck.sh` / `deploy.sh` (see
[roadmap.md](./roadmap.md#next)). "Did this deploy succeed?" is a readiness question, and
asking it through the proxy exercises the whole chain (nginx → FastCGI → Laravel →
Postgres/Redis/RabbitMQ) rather than asking one container about itself. Beyond that, these routes are
the standard shape for an external uptime monitor, a load balancer deciding whether to route
to a node, or Kubernetes `livenessProbe`/`readinessProbe`. None of those exist here yet, and
the first two would need the loopback ACL relaxed.

## `location /api/` → `fastcgi_pass api:9000`

This is the one non-obvious part of the whole config, worth understanding properly.

`laravel-shop-api`'s image runs `php-fpm` and exposes port `9000` — php-fpm speaks
**FastCGI**, a binary protocol, not HTTP. `proxy_pass` only knows how to speak HTTP/HTTPS;
it cannot talk to php-fpm at all, with or without a port number. `fastcgi_pass` is the
directive built for this.

`SCRIPT_FILENAME` is hardcoded to Laravel's front controller
(`/var/www/html/public/index.php`, matching the path set by `laravel-shop-api`'s
Dockerfile). This is necessary because `proxy` and `api` are **separate containers with no
shared filesystem** — nginx has no copy of Laravel's source to `try_files` against, so
every request is forced through the one entry point. This is exactly how Laravel already
expects to be run (all HTTP requests go through `public/index.php` regardless of route),
so nothing is lost by doing it this way.

`include fastcgi_params;` is **not a file you need to create** — it ships inside the
nginx image itself (`/etc/nginx/fastcgi_params`) and supplies the standard set of
FastCGI parameters (`REQUEST_URI`, `QUERY_STRING`, `REQUEST_METHOD`, `REDIRECT_STATUS`,
etc.). Laravel's router specifically needs `REQUEST_URI` to route correctly, and php-fpm
requires `REDIRECT_STATUS` to be set at all (a built-in security check) — replacing this
line with only a handful of hand-picked `fastcgi_param` lines would silently break both.

### This location is a contract — don't rename it casually

Three separate pieces of software agree on the string `/api`, and nothing but convention
enforces it:

| Party | What it assumes |
|---|---|
| The frontend bundle | the API lives at `/api` on whatever origin served the page (`ARG VITE_API_URL=/api`, baked in at build time) |
| This file | `location /api/` → `fastcgi_pass api:9000` |
| Laravel | `withRouting(api: …)` puts every API route under `/api` |

Changing it means changing all three, and publishing a new frontend image. The dangerous part
is how a partial change *fails*: rename this location alone and nginx still starts, and
**every container healthcheck stays green**. None of them sends a request through `/api/`.
Meanwhile every browser API call falls through to `location /` and lands on the React
container, which answers with its `index.html` (or a 405 for a POST). The whole application
is broken, and no health check notices.

The only thing that catches it is a request that actually goes through `/api/` and checks
that Laravel answered: an unauthenticated `curl http://localhost/api/user` should return a
401 *in JSON*. See [hands-on.md](./hands-on.md#touch-each-service-directly) for the manual
check, and how to read what comes back. Automating it is a natural job for the planned
`healthcheck.sh`.

No prefix stripping happens here on purpose: `laravel-shop-api` registers `routes/api.php`
with Laravel's own automatic `/api` prefix (see `bootstrap/app.php`'s
`withRouting(api: ...)`), so a client request to `/api/products` needs to reach Laravel as
`/api/products`, unchanged — which is exactly what happens, since `$request_uri` (set by
`fastcgi_params`) always carries the original, unmodified path.

## HTTPS (`443`)

`compose.yml` publishes port `443`, but this file has no `listen 443 ssl` server block yet,
and won't until two prerequisites exist:

1. **A real registered domain** pointing (DNS `A` record) at the server's public IP — a
   certificate authority like Let's Encrypt cannot issue a certificate for `localhost` or a
   bare IP address.
2. **Actual certificate files.** nginx refuses to even start if a `listen ... ssl` block is
   present without a matching `ssl_certificate`/`ssl_certificate_key` — so this can't be
   added speculatively without breaking the whole `proxy` container.

This is intentionally deferred until there's a real server + domain to issue a certificate
for (see [roadmap.md](./roadmap.md)).

## Further reading

- nginx `fastcgi_pass`/`fastcgi_param` reference: https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html
- What FastCGI is, and why php-fpm speaks it instead of HTTP: https://www.php.net/manual/en/install.fpm.php
- Laravel's own deployment/server notes: https://laravel.com/docs/12.x/deployment
