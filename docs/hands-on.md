# Hands-on: validating and exploring the stack

Related: [compose.md](./compose.md) · [nginx.md](./nginx.md) · [environment.md](./environment.md) · [README](../README.md)

Commands to check things *before* starting, and to poke at them by hand *after*. Every
command here is copy-pasteable from the repo root.

**Working locally?** Start the session with:

```bash
export COMPOSE_FILE=compose.yml:compose.dev.yml
```

That makes the plain `docker compose …` commands below include the local port exposure from
`compose.dev.yml`. Without it you'd pass `-f compose.yml -f compose.dev.yml` every time.
On a server, export nothing — the bare command is already the production-safe one.

## Before `up` — validate without starting anything

**Compose file resolves and is valid:**
```bash
docker compose config --quiet && echo OK
```
Silence plus `OK` means valid. No `--env-file` flag is needed — Compose reads `.env` from
the project directory automatically.

⚠️ **Check for warnings, not just the exit code.** If `.env` is missing or a variable is
absent from it, Compose does *not* fail: it prints `variable is not set, defaulting to a
blank string` and happily produces a config with empty passwords and an image tag of `:`.
Read the output rather than trusting the `OK`.

**See what one service will actually get** (the resolved values, secrets included):
```bash
docker compose config api
docker compose config --services          # just list service names
```

**nginx config syntax:**
```bash
docker run --rm --add-host frontend:127.0.0.1 --add-host api:127.0.0.1 \
  -v "$PWD/proxy/default.conf:/etc/nginx/conf.d/default.conf:ro" \
  nginx:1.31-alpine nginx -t
```
The `--add-host` flags matter: nginx resolves upstream hostnames **at startup**, so without
them the test dies with `host not found in upstream "frontend"` — a DNS failure, not a
syntax error. (That same startup-resolution behaviour is why a redeployed backend container
with a new IP needs an `nginx -s reload`.)

**Catch a bad image tag before it wastes a deploy:**
```bash
docker compose pull
```

**Port 80 is actually free:**
```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/   # want: connection refused (000)
docker ps --format '{{.Names}} {{.Ports}}' | grep -E ':(80|443)->'   # want: nothing
```
Any HTTP status at all means something already answers on port 80, and after `up` your
browser and `curl` would be talking to *it*, not to this stack. Don't rely
on `ss -ltn` alone. Kubernetes tools like k3s expose ports through firewall NAT rules rather
than a listening socket, so `ss` shows nothing while `curl` still gets an answer. (On this
repo's development machine, that turned out to be k3s's Traefik answering
`404 page not found`.)

## Getting an image to run

`API_VERSION` and `FRONTEND_VERSION` can only point at tags that were actually published to
GHCR. Normally that means a released version — push a git tag `v0.1.2` in the app repo and
its `docker-publish.yml` publishes `0.1.2`.

To run something that **isn't released yet** — a feature branch, or just to check a change
works before committing to a release — use the manual trigger instead: in the app repo,
**Actions → Publish Docker image → Run workflow**, choosing the branch you want. The
workflow accepts `workflow_dispatch` for exactly this, and since a manual run has no semver
tag behind it, the image is published under the bare 7-character **commit SHA**:

```bash
API_VERSION=a3f9c21        # in .env, works like any other tag
```

Two things this does *not* compromise: the test suite still gates the build (the workflow
calls `ci.yml` first), and `latest` doesn't move, since that only follows semver tag pushes.

Use this for verification. Production should be pinned to semver tags — those are the ones
with release intent behind them.

## First start — from an empty database to a working login

Every step in order. Assumes `.env` is filled in (see the README) and, locally, that
`COMPOSE_FILE=compose.yml:compose.dev.yml` is exported.

**1. Start everything**
```bash
docker compose up -d
docker compose ps            # all six should reach (healthy)
```

**2. Migrations: already done**

The `migrate` service ran them before `api` was allowed to start. Confirm:
```bash
docker compose logs migrate  # expect "Running migrations" and a list of DONE lines
```

**3. Seed data: not available in this image**

`php artisan db:seed` fails here with `Call to undefined function Database\Factories\fake()`.
The seeders use factories, factories need Faker, and Faker is a dev-only (`require-dev`)
package that the production image doesn't install. To start with real data, restore a dump
instead (see [Database: export, update, recreate](#database-export-update-recreate)), or
create records through the API.

**4. Create a customer**
```bash
curl -s -X POST http://localhost/api/admin/register \
  -H 'Accept: application/json' -H 'Content-Type: application/json' \
  -d '{"name":"Customer","email":"customer@example.com","password":"secret-pass-123","password_confirmation":"secret-pass-123"}'
```
Despite the `/admin/` in its URL, this endpoint creates a **customer**: the `system_role`
column defaults to `customer`, and `register` never sets it.

**5. Create an admin:** register, then promote
```bash
curl -s -X POST http://localhost/api/admin/register \
  -H 'Accept: application/json' -H 'Content-Type: application/json' \
  -d '{"name":"Admin","email":"admin@example.com","password":"secret-pass-123","password_confirmation":"secret-pass-123"}'

docker compose exec api php artisan tinker --execute="App\Models\User::where('email','admin@example.com')->update(['system_role'=>'admin']);"
```

**6. Log in and make an authenticated request**
```bash
TOKEN=$(curl -s -X POST http://localhost/api/admin/login \
  -H 'Accept: application/json' -H 'Content-Type: application/json' \
  -d '{"email":"admin@example.com","password":"secret-pass-123"}' \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["data"]["token"])')

curl -s http://localhost/api/user -H 'Accept: application/json' -H "Authorization: Bearer $TOKEN"
```
The second call should return the admin user as JSON. The stack works end to end.

⚠️ **Two things to fix in `laravel-shop-api` before any public deploy:**
`/api/admin/register` is open to anyone, and `/api/admin/login` checks no role, so any
registered customer can log in to the admin endpoint and get a token. That's fine on your
laptop, and a real hole on a server.

## After `up` — see it actually working

```bash
docker compose up -d
```

**Status and health of everything:**
```bash
docker compose ps
```
The `STATUS` column shows `(healthy)` / `(starting)` / `(unhealthy)`.

**Why did a healthcheck fail?** This is the one to remember — Docker stores the last few
probe runs, with their output:
```bash
docker inspect --format '{{json .State.Health}}' shop-api-1 | jq
```

**Logs:**
```bash
docker compose logs -f api          # one service, follow
docker compose logs --tail=50       # everything, recent
```

**Get inside a container:**
```bash
docker compose exec api sh          # runs as www-data
docker compose exec -u root api sh  # if you need root
```

**Run the readiness probe by hand** — the same command Docker runs every 30s:
```bash
docker compose exec api php artisan health:check; echo "exit=$?"
```
`exit=0` is healthy, `exit=1` unhealthy. The printed `database: ok` lines are for you;
Docker only reads that exit code.

**Has the database schema been created?**
```bash
docker compose exec api php artisan migrate:status
```
Normally the one-shot `migrate` service has already done this on `up` (see
[compose.md](./compose.md#the-migrate-service)). To see what it ran, or run it again:
```bash
docker compose logs migrate          # what it applied on the last up
docker compose run --rm migrate      # run pending migrations now ("Nothing to migrate" if none)
```
`docker compose ps` won't list `migrate`, because it has exited; `docker compose ps -a` shows
`Exited (0)`, which means it succeeded. `Migration table not found` from `migrate:status`
means migrations have never run. **No health check catches this**: readiness only proves
Postgres is *reachable*, not that it has tables. The symptom is every real API call returning
`500` while every health check passes.

**Inside a container, use `127.0.0.1`, not `localhost`.**
```bash
docker compose exec frontend wget -q --spider http://127.0.0.1/ && echo OK   # works
docker compose exec frontend wget -q --spider http://localhost/ && echo OK   # connection refused
```
Inside these Alpine containers `localhost` resolves to the IPv6 address `::1` first, but both
nginx configs here (the proxy's and the frontend image's) listen on IPv4 only. The official
nginx image adds IPv6 listening automatically, but only to its *stock* config, never to a
custom one. That's why every in-container probe in this repo uses `127.0.0.1`. From the
**host**, `http://localhost` is fine, because Docker forwards both address families.

## Touch each service directly

**The network — who's on it, and does DNS work:**
```bash
docker network inspect shop_default --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
docker compose exec api getent hosts postgres rabbitmq redis
docker compose exec api nc -zv postgres 5432
```
Service names resolve to container IPs via Docker's internal DNS — this is the "Path 1"
networking described in [compose.md](./compose.md#why-two-files-instead-of-one).

**Postgres:**
```bash
docker compose exec postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "\dt"'
```
The `sh -c '…'` with **single quotes** matters. `$POSTGRES_USER` exists only *inside* the
container. Written without it, your host shell expands the variables first, finds them empty,
and psql fails with `role "root" does not exist`. The single quotes stop the host from
touching them, so the container's shell expands them instead. Every `psql`/`pg_dump` command
in this doc follows that pattern.

**Redis** — note the cache lives in logical DB **1**, not 0:
```bash
docker compose exec redis redis-cli ping
docker compose exec redis redis-cli -n 1 keys '*'
```

**RabbitMQ:**
```bash
docker compose exec rabbitmq rabbitmqctl list_users      # expect shop_user, not guest
docker compose exec rabbitmq rabbitmqctl list_vhosts
docker compose exec rabbitmq rabbitmqctl list_queues
```
Or the web UI at http://localhost:15672 (local only — see
[environment.md](./environment.md#software-for-the-locally-exposed-ports)).

**End to end through the proxy:**
```bash
curl -i http://localhost/                 # → the React app
curl -i http://localhost/api/user         # → 401, Content-Type: application/json
curl -i http://localhost/health/ready     # → 403: the ACL works
docker compose exec proxy wget -qO- http://127.0.0.1/health/ready   # → the JSON
```
The `/api/user` line is the one that tests the wiring. There's no public `GET` under `/api`,
so it asks for the current user without a token. A **401 in JSON** can only come from
Laravel: it proves nginx routed the request to FastCGI and the app booted, matched the route
and ran its auth middleware. It has no side effects, so it's safe to run any time. Look at
the **`Content-Type`**, not just the status: `application/json` means Laravel answered;
`text/html` means the request fell through to the React container, which is how a broken
`/api/` route shows up (see
[nginx.md](./nginx.md#this-location-is-a-contract--dont-rename-it-casually)).

That 403-from-outside / JSON-from-inside pair on `/health/ready` is the loopback restriction
doing its job.

**Reading what you get back** when you run `curl` by hand:

| You see | On | It means |
|---|---|---|
| `200`, `text/html` | `/` | ✅ frontend served through the proxy |
| `401`, `application/json` | `/api/user` | ✅ Laravel answered: routing, FastCGI and app all work |
| `200`/`405`, **`text/html`** | `/api/…` | ❌ fell through to the frontend: `location /api/` isn't routing to php-fpm |
| `500`, JSON `Internal server error` | `/api/…` | ❌ Laravel is running but failing: check `migrate:status` above, then `docker compose logs api` |
| `502 Bad Gateway` | anything | ❌ proxy is up, backend isn't answering: container down, **or** nginx still has the old IP of a recreated container (`docker compose exec proxy nginx -s reload`) |
| `504 Gateway Timeout` | anything | ❌ backend too slow: hit `fastcgi_read_timeout` |
| `403` | `/health/*` from the host | ✅ expected: those routes are loopback-only |
| connection refused / `000` | anything | ❌ proxy not running, or wrong host/port |
| `404 page not found` (plain text) | anything | ❌ not this stack at all: something else owns port 80 (see *Before `up`*) |

Always look at the **Content-Type**, not just the status. It's what tells Laravel's
answers apart from the frontend's.

## Database: export, update, recreate

Dumps go in `backups/`, which is git-ignored: a dump contains every email and password hash
in the database, so treat it like `.env`.

**Export**
```bash
mkdir -p backups
docker compose exec -T postgres sh -c 'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" --clean --if-exists --no-owner' \
  > backups/shop-$(date +%F).sql
```
- `-T` stops Docker allocating a terminal, which would corrupt output redirected to a file.
- `--clean --if-exists` writes a `DROP … IF EXISTS` before each `CREATE`, which is what lets
  this same file be loaded over an existing database (next section).
- `--no-owner` leaves out ownership, so the dump restores under any database user.

**Update an existing database from a dump**: replaces every table in the dump, and leaves
anything that isn't in it alone:
```bash
docker compose stop api
docker compose exec -T postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -v ON_ERROR_STOP=1' \
  < backups/shop-2026-09-26.sql
docker compose start api
```

**Recreate the database from a dump**: throws the whole database away first, so nothing
from before survives:
```bash
docker compose stop api
docker compose exec -T postgres sh -c 'dropdb -U "$POSTGRES_USER" --force "$POSTGRES_DB" && createdb -U "$POSTGRES_USER" "$POSTGRES_DB"'
docker compose exec -T postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -v ON_ERROR_STOP=1' \
  < backups/shop-2026-09-26.sql
docker compose start api
```

Notes that apply to both:
- **Stop `api` first**, so nobody is served half-restored data. Meanwhile the proxy returns
  `502` for API calls, which is expected.
- **`ON_ERROR_STOP=1`** makes psql stop at the first error instead of carrying on and leaving
  a partially loaded database that *looks* fine.
- **`dropdb --force`** disconnects anything still connected; without it, the drop fails
  while any session is open.
- **The dump includes Laravel's `migrations` table**, so after a restore the `migrate`
  service knows what's already applied, and on the next `up` runs only migrations newer than
  the dump.
- **The nuclear option** is `docker compose down -v`, which deletes the Postgres volume
  (along with Redis's and RabbitMQ's). The next `up` gives you an empty database and
  `migrate` builds the schema again.

## Break it on purpose

The fastest way to feel how the layers connect:

```bash
docker compose stop rabbitmq
docker compose exec api php artisan health:check; echo "exit=$?"   # rabbitmq: error, exit=1
docker compose ps                                                  # api flips to (unhealthy)
docker compose start rabbitmq                                      # recovers on the next probe
```

Then try `docker compose stop frontend` and watch `curl http://localhost/` start returning
`502` — nginx is up and healthy (its own `/nginx-health` still passes), but the thing behind
it is gone. That's exactly why the proxy's healthcheck deliberately doesn't test its
backends.

## Cleaning up

```bash
docker compose down        # stop and remove containers, keep data
docker compose down -v     # ...and delete the volumes (wipes the database)
```
