# Hands-on: validating and exploring the stack

Related: [compose.md](./compose.md) · [nginx.md](./nginx.md) · [environment.md](./environment.md) · [README](../README.md)

Commands to check things *before* starting, and to poke at them by hand *after*. Every
command here is copy-pasteable from the repo root.

## Before `up` — validate without starting anything

**Compose file resolves and is valid:**
```bash
docker compose --env-file .env.prod config --quiet && echo OK
```
Silence plus `OK` means valid. ⚠️ **Always pass `--env-file`.** Forget it and Compose does
not fail — it prints `variable is not set, defaulting to a blank string` warnings and
happily produces a config with empty passwords and an image tag of `:`.

**See what one service will actually get** (the resolved values, secrets included):
```bash
docker compose --env-file .env.prod config api
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
docker compose --env-file .env.prod pull
```

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
API_VERSION=a3f9c21        # in .env.prod, works like any other tag
```

Two things this does *not* compromise: the test suite still gates the build (the workflow
calls `ci.yml` first), and `latest` doesn't move, since that only follows semver tag pushes.

Use this for verification. Production should be pinned to semver tags — those are the ones
with release intent behind them.

## After `up` — see it actually working

```bash
docker compose --env-file .env.prod up -d
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

## Touch each service directly

**The network — who's on it, and does DNS work:**
```bash
docker network inspect shop_default --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
docker compose exec api getent hosts postgres rabbitmq redis
docker compose exec api nc -zv postgres 5432
```
Service names resolve to container IPs via Docker's internal DNS — this is the "Path 1"
networking described in [compose.md](./compose.md#why-two-files-instead-of-one).

**Postgres** (the env vars are already set inside the container):
```bash
docker compose exec postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c '\dt'
```

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
curl -i http://localhost/api/admin/login  # → Laravel JSON (405 for GET is fine — it routed)
curl -i http://localhost/health/ready     # → 403: the ACL works
docker compose exec proxy wget -qO- http://localhost/health/ready   # → the JSON
```
That 403-from-outside / JSON-from-inside pair is the loopback restriction doing its job.

**Prove the local/production port split:**
```bash
docker compose ps --format '{{.Service}}\t{{.Ports}}'
```
With the override merged you'll see `5432`, `6379`, `5672`, `15672` published. Run
`docker compose -f compose.yml ... ps` instead and only `proxy` has published ports.

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
