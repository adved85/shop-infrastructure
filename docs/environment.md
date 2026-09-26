# Environment files: `.env.example` and `.env`

Related: [compose.md](./compose.md) · [nginx.md](./nginx.md) · [roadmap.md](./roadmap.md) · [hands-on.md](./hands-on.md) · [README](../README.md)

## `.env.example` vs `.env`

`.env.example` is committed to git — it documents every variable the stack needs, with
placeholder values for anything secret. `.env` is where the *real* values live, and it
is deliberately **git-ignored** (see `.gitignore`) — it should never be committed, since it
holds real database and message-broker passwords.

To set up either environment (local or production), the process is the same:

```bash
cp .env.example .env
# then edit .env and replace every CHANGE_ME with a real generated secret:
openssl rand -base64 32
```

Compose reads `.env` from the project directory **automatically**, so no `--env-file` flag
appears in any command here. (A differently-named file — `.env.prod`, say — would need
`--env-file` passed every single time.)

### Why it's `.env` and not `.env.prod`

Because it isn't the *production* file — it's **this machine's** file. It's git-ignored, so
every host has its own copy: your laptop's holds throwaway secrets, the server's holds real
ones, and neither ever travels. Naming it after one environment would misdescribe both.

What actually distinguishes local from production is which **compose files** you combine
(see [compose.md](./compose.md#why-two-files-instead-of-one)), not which variables you set —
the variable *names* are identical everywhere, only the values differ per machine.

There's also no separate `.env.dev`. One would mainly earn its keep if you wanted faster
local iteration (pointing at a `latest`/locally-built image instead of a pinned release) or
Laravel's debug mode enabled locally — neither has been needed yet.

## Variable reference

| Variable | Used by | Notes |
|---|---|---|
| `APP_KEY` | `api` | Laravel's encryption key. See the warning below — this one behaves differently from the other secrets. |
| `POSTGRES_DB` | `postgres`, `api` | Database name. |
| `POSTGRES_USER` | `postgres`, `api` | Must match what the healthcheck (`pg_isready -U ...`) expects — the two are wired to the same variable, so they can't drift. |
| `POSTGRES_PASSWORD` | `postgres`, `api` | Generate with `openssl rand -base64 32`. Never reuse a password that's already been pasted somewhere insecure (chat, a screenshot, a public log) — treat it as compromised and rotate it. |
| `REDIS_HOST` / `REDIS_PORT` | `api` | Points at the `redis` service over the internal Docker network. |
| `RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS` | `rabbitmq`, `api` | Same rotation advice as the Postgres password. See the section below — these two names carry more weight than they look like they do. |
| `API_VERSION` / `FRONTEND_VERSION` | `api`, `frontend` | See below. |

## `APP_KEY` — the one that fails silently

Laravel's encryption key, and the only secret here that can be missing without anything
appearing to be wrong. Tested against the real image with no `APP_KEY` set:

```
status=running                 ← the container starts normally
ready to handle connections    ← php-fpm is serving

config('app.key')  →  NULL
app('encrypter')   →  MissingAppKeyException: No application encryption key has been specified.
```

The container boots, php-fpm listens, the entrypoint's `config:cache` succeeds — and
**`php artisan health:check` passes too**, because readiness probes Postgres, Redis and
RabbitMQ and none of them resolve the encrypter. So a missing `APP_KEY` gives you six green
containers and an application that throws the moment anything needs encryption. That's the
one failure mode the health checks in this stack cannot see.

Generate one with:

```bash
echo "base64:$(openssl rand -base64 32)"
```

**It is not rotatable like the other passwords.** A database password can be changed at will;
`APP_KEY` is the key that *decrypts existing data*. Change it and anything encrypted under
the old key — encrypted columns, signed URLs, password-reset tokens, encrypted cookies —
becomes permanently unreadable. Generate it once per deployment and leave it alone.

One thing the image gets right here: the entrypoint runs `config:cache` on **every**
container start rather than at build time, so the key present in the environment at startup
is the one that ends up in the cached config. A key added later is picked up on the next
restart, with no stale cached `null` left behind.

## Telling the app to actually *use* the services

Starting Redis and RabbitMQ and handing the app their connection details is only half the
job — it makes them *reachable*, not *used*. Which driver Laravel actually picks is a
separate decision, made by two more variables, and both of them default to `database`:

```php
// config/cache.php        'default' => env('CACHE_STORE', 'database'),
// config/queue.php        'default' => env('QUEUE_CONNECTION', 'database'),
```

Left unset, you'd get a stack that runs Redis and RabbitMQ, successfully verifies both in
the readiness probe, and then quietly routes every cache write and every queued job to
Postgres anyway. Nothing errors; the services just sit idle. So `compose.yml` sets both
explicitly on `api`:

```yaml
CACHE_STORE: redis
QUEUE_CONNECTION: rabbitmq
```

Two details worth knowing about how those resolve:

- **`CACHE_STORE: redis` uses a different Redis connection than you might expect.**
  `config/cache.php`'s redis store points at the `cache` connection (not `default`), which
  reads the same `REDIS_HOST`/`REDIS_PORT` we already pass but uses Redis logical database
  **1**, while the `default` connection uses database **0**. No extra variables needed —
  just be aware that cached values and anything written through `Redis::` directly live in
  separate keyspaces on the same server. (It also means the readiness probe, which pings
  the `default` connection, is verifying the same server the cache uses, just a different
  logical DB.)
- **`REDIS_PASSWORD` is deliberately not passed.** Our Redis runs without `requirepass`, and
  Laravel's config defaults the password to `null`, which is the matching setting.

`SESSION_DRIVER` is intentionally *not* set, so sessions fall back to `database` (Postgres).
That's a sensible default for a token-authenticated API, and it's precisely why the health
routes are registered with no middleware — under the `web` group they'd open a session and
make every probe depend on Postgres.

## `VITE_API_URL` — why it isn't here

The frontend's API base URL is the one setting you might expect in `.env` and won't
find. It's baked into the JavaScript bundle at **image build time**, so by the time this
repo pulls the image it's already fixed. No environment variable here can change it.

Where it comes from is the Dockerfile in `react-shop-client` itself:

```dockerfile
ARG VITE_API_URL=/api
```

It used to be supplied by a GitHub repository variable. That was dropped for a good reason:
the CI smoke-test build never passed the variable while the publish build did, so the image
CI verified and the image that got published were built differently. With the default in
the Dockerfile and nobody overriding it, both builds are identical. One trap is noted in
that Dockerfile: passing an *empty* `--build-arg VITE_API_URL=` replaces the default rather
than falling back to it, and ships a bundle with no API URL at all.

Its value is the relative path **`/api`**, deliberately, and that choice is what lets one
published image serve every environment. The browser resolves a relative path against
whatever origin served the page:

| Page origin | Resulting API call |
|---|---|
| `http://localhost` | `http://localhost/api/...` |
| `https://yourdomain.com` | `https://yourdomain.com/api/...` |

Both land where the proxy already routes `/api/` to Laravel. An absolute URL would instead
pin the image to a single environment and force a separate build per environment — and,
being cross-origin, would require CORS configuration on the Laravel side. Neither is needed
here, which is why there is no `config/cors.php` in the API repo.

The catch worth remembering: this holds only while the frontend and API share an origin. If
they're ever split onto separate domains, the relative path stops working and CORS becomes
necessary.

## RabbitMQ credentials: two sets of names for one account

This one caused a real bug, so it's worth understanding rather than copying.

There are **two different variable namespaces** involved, belonging to two different pieces
of software:

| Read by | Variables | Purpose |
|---|---|---|
| The **broker** (`rabbitmq` container) | `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`, `RABBITMQ_DEFAULT_VHOST` | Creates the initial account when the broker first starts |
| The **app** (`api` container) | `RABBITMQ_HOST`, `RABBITMQ_PORT`, `RABBITMQ_USER`, `RABBITMQ_PASSWORD`, `RABBITMQ_VHOST` | How Laravel's queue package connects *as* that account |

The names are not interchangeable, and nothing warns you when they're wrong. Passing only
`RABBITMQ_DEFAULT_USER`/`RABBITMQ_DEFAULT_PASS` to `api` — which looks perfectly reasonable —
means Laravel reads *none* of them and silently falls back to its own defaults:
`RABBITMQ_HOST` → `127.0.0.1` (the api container dialling itself instead of the broker) and
`guest`/`guest` for credentials (an account that doesn't exist once you've set a default
user, and which RabbitMQ refuses from non-loopback connections anyway). The failure surfaces
only as `rabbitmq: error` in the readiness probe.

So `compose.yml` passes the app-side names explicitly, mapping the credentials from the
*same two variables* the broker uses:

```yaml
api:
  environment:
    RABBITMQ_HOST: rabbitmq
    RABBITMQ_PORT: 5672
    RABBITMQ_VHOST: /
    RABBITMQ_QUEUE: default
    RABBITMQ_USER: ${RABBITMQ_DEFAULT_USER}
    RABBITMQ_PASSWORD: ${RABBITMQ_DEFAULT_PASS}
```

That mapping is the important part: there is exactly **one** place a credential is defined
(`.env`), so the broker's account and the app's login cannot drift apart. Defining a
second pair of variables for the app side would work right up until somebody rotated one and
not the other.

Note also that `.env` holds only the two `RABBITMQ_DEFAULT_*` credentials. Host, port,
vhost and queue name are hardcoded in `compose.yml` instead, because they're topology
constants — identical in every environment — exactly like `DB_HOST: postgres` and
`DB_PORT: 5432`. Only secrets and per-deployment choices belong in the env file.

### `host` vs. `vhost` — completely different things

Easy to conflate because the names look related. They aren't:

- **`RABBITMQ_HOST=rabbitmq`** is a *network address* — which machine to open a TCP
  connection to. Here it's the Compose service name, resolved by Docker's internal DNS to
  the broker container's IP. Same category of thing as `DB_HOST=postgres`.
- **`RABBITMQ_VHOST=/`** is a *virtual host* — a named namespace **inside** the broker,
  after you've already connected and authenticated. Each vhost has its own queues,
  exchanges, bindings and permissions, fully isolated from other vhosts. It's closest in
  spirit to a database *name* in Postgres: one server, several independent logical spaces
  inside it.

The `/` value is RabbitMQ's default vhost, which is what a single-application setup
normally uses. Multiple vhosts start earning their keep when one broker serves several
applications (or a staging and production workload) and you want their queues and
permissions kept entirely separate — a user granted access to `/shop` simply cannot see
anything in `/analytics`.

## `API_VERSION` / `FRONTEND_VERSION`

These pin the exact image tag pulled from GHCR for `api` and `frontend`
(`ghcr.io/adved85/laravel-shop-api:${API_VERSION}`, etc.). They are **not** arbitrary —
each value must correspond to a tag that was actually published by that repo's own CI.

Both `laravel-shop-api` and `react-shop-client` publish images via a `docker-publish.yml`
GitHub Actions workflow, triggered by pushing a git tag matching `v*.*.*` (e.g. `v0.1.0`).
That workflow first re-runs the repo's own `ci.yml` (tests + a build smoke-test) via
`workflow_call`, and only pushes the image to GHCR if that passes — so a version referenced
here is guaranteed to have been tested before being published, not just built.

- API releases: https://github.com/adved85/laravel-shop-api/releases (or `/tags`)
- Frontend releases: https://github.com/adved85/react-shop-client/releases (or `/tags`)
- Published images: https://github.com/adved85?tab=packages

Two kinds of value are valid here: a **semver version** (`0.1.0`) from a tagged release, or
a bare 7-character **commit SHA** (`a3f9c21`) from a manually dispatched build — see
[hands-on.md](./hands-on.md#getting-an-image-to-run) for how to produce one. Use SHA tags to
verify something unreleased; keep production pinned to semver, which carries release intent.

## Why each service only sees the variables it needs

`compose.yml` does **not** use a blanket `env_file: .env` on every service. Each
service's `environment:` block explicitly lists only the variables it actually needs:

```yaml
postgres:
  environment:
    POSTGRES_DB: ${POSTGRES_DB}
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

This matters for blast radius: with a blanket `env_file:`, every container — including
`frontend` — would receive every secret in the file, whether it needed it or not. Under
this setup, if any *one* container were ever compromised, an attacker could read every
credential in the stack from its environment, not just the one that container legitimately
uses. `frontend` in particular needs **zero** runtime environment variables at all — its
API URL is baked into the build at image build time (see
[nginx.md](./nginx.md#location--→-frontend80)), so it has no reason to ever hold a
database or broker password.

## Local vs. production ports

`.env`'s variables never change between local and production use — what changes is
which `docker compose` invocation you run:

```bash
# local — opt in to compose.dev.yml, exposing
# 5432 / 6379 / 5672 / 15672 on localhost for GUI tools
docker compose -f compose.yml -f compose.dev.yml up -d

# production — the plain command, so only 80 / 443 (proxy)
# are ever reachable from outside the server
docker compose up -d
```

The reasoning: `api` reaches Postgres/Redis/RabbitMQ over the internal Docker network by
service name (`postgres:5432`, etc.) regardless of whether a host port is published —
`ports:` only controls host-machine/internet reachability, which is a real risk on a public
server but harmless on your own laptop. Full explanation and the two-path mental model are
in [compose.md](./compose.md#why-two-files-instead-of-one).

### Software for the locally-exposed ports

With `compose.dev.yml` included, these are reachable from your own machine:

| Port | Service | Suggested tools |
|---|---|---|
| `5432` | Postgres | TablePlus, DBeaver, pgAdmin |
| `6379` | Redis | RedisInsight, Redis Commander, `redis-cli` |
| `5672` | RabbitMQ (AMQP) | Used by application code, not something you browse directly |
| `15672` | RabbitMQ management UI | Open http://localhost:15672 in a browser and log in with `RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS`. Works because the image is pinned to `rabbitmq:4-management` — the plain `rabbitmq` image does **not** enable the `rabbitmq_management` plugin, so this port would answer nothing. |

## Local-only: debug mode

`compose.dev.yml` also switches the API into debug mode, locally only:

```yaml
api:
  environment:
    APP_ENV: local
    APP_DEBUG: "true"
```

Compose merges these keys into `api`'s existing `environment:` block. They have to sit under
`environment:`; placed directly under `api:` they're rejected as unknown service properties.
`"true"` is quoted so it stays the string Laravel expects rather than becoming a YAML boolean.

**What it buys you:** real error messages. With debug off, a failure comes back as
`{"message":"Internal server error"}`, and the logs are the only place that says why. With it
on, the response includes the actual exception and a short trace, so a missing table shows
up as *relation "users" does not exist* right in your `curl` output.

**Why it can't leak:** production never loads `compose.dev.yml`, so there `APP_ENV` stays at
the image's built-in `production` and `APP_DEBUG` stays off. Error detail can include file
paths, SQL and configuration, which is exactly what you want locally and never on a public
server.

**The trade-off, honestly stated:** this is a deliberate exception to "the dev file only adds
ports". `APP_ENV=local` changes more than error output; for example, `artisan migrate` no
longer asks for confirmation, so `--force` becomes unnecessary. So locally you run a slightly
different configuration from production. Error reporting is the one difference worth that
cost. Behaviour that changes what the app actually *does* should stay identical in both.
