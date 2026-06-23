# Docker Compose

**What it is:** A tool to define and run **multi-container** apps from a single `compose.yaml` file.

**Why people use it:** One file + one command (`docker compose up`) spins up your whole stack (app + db + cache + …) with networking, volumes, and env wired together — instead of juggling many `docker run` commands.

**Typically used for:** Local dev environments, integration tests in CI, and simple single-host deployments.

> Note: `docker compose` (v2, a plugin, space) replaces the old `docker-compose` (v1, Python, hyphen). Use the space form.

---

## Most common commands

```bash
docker compose up                  # start all services (foreground)
docker compose up -d               # start detached (background)
docker compose up --build          # rebuild images first, then start
docker compose down                # stop & remove containers + network
docker compose down -v             # also remove named volumes (wipes data)
docker compose ps                  # list services in this project
docker compose logs -f             # tail logs from all services
docker compose logs -f api         # tail one service
docker compose exec api sh         # shell into a running service
docker compose run --rm api npm test   # one-off command in a fresh container
docker compose build               # build/rebuild images
docker compose restart api         # restart one service
docker compose pull                # pull updated images
```

## Simple usage

A web app + Postgres in one file:

```yaml
# compose.yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/app
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - dbdata:/var/lib/postgresql/data

volumes:
  dbdata:
```

```bash
docker compose up -d      # start everything
docker compose logs -f    # watch it boot
docker compose down       # tear it all down
```

Note `api` reaches the database at host `db` — **Compose makes service names resolvable** on a shared network.

---

## Advanced

### Healthchecks + `depends_on` conditions

`depends_on` alone only waits for the container to *start*, not to be *ready*. Gate startup on a healthcheck so `api` waits until Postgres actually accepts connections:

```yaml
services:
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
```

### Hot-reload dev setup (bind mounts)

Mount source into the container so edits reflect immediately, while keeping container-installed `node_modules`:

```yaml
services:
  api:
    build: .
    volumes:
      - .:/app
      - /app/node_modules     # anonymous volume shields container's deps
    command: npm run dev
```

### Develop watch (live sync / rebuild)

The modern alternative to bind-mounting — `docker compose watch` reacts to file changes per-path, without restarting by hand:

```yaml
services:
  web:
    build: .
    develop:
      watch:
        - action: sync          # copy changed files into the container (hot-reload)
          path: ./src
          target: /app/src
        - action: sync+restart  # sync, then restart the service (config changes)
          path: ./config.yaml
          target: /app/config.yaml
        - action: rebuild       # rebuild the image (e.g. package.json changed)
          path: ./package.json
```

```bash
docker compose watch
```

### Networks & aliases

All services share one default network where **service names are DNS hostnames**. Define custom networks to segment (e.g. frontend/backend):

```yaml
services:
  web: { build: ., networks: [frontend] }
  api: { build: ./api, networks: [frontend, backend] }
  db:  { image: postgres, networks: [backend] }   # web cannot reach db
networks:
  frontend: {}
  backend: {}
```

- **aliases** — extra DNS names for a service: `networks: { backend: { aliases: [postgres-primary] } }`.
- **links** (legacy) — predates Compose networks; unnecessary now since service names already resolve.

### Override files (dev vs prod)

Compose auto-merges `compose.override.yaml` on top of `compose.yaml`. Keep prod base clean, layer dev-only tweaks in the override:

```bash
# Uses compose.yaml + compose.override.yaml automatically
docker compose up

# Explicit file set, e.g. for CI/prod
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

### `.env` and variable substitution

Compose auto-loads a `.env` file in the project dir:

```yaml
services:
  db:
    image: postgres:${PG_VERSION:-16}    # default to 16 if unset
    ports:
      - "${DB_PORT:-5432}:5432"
```

```bash
# .env
PG_VERSION=15
DB_PORT=5433
```

### Profiles (optional services)

Start extra services only when asked — e.g. a debug UI or seed job:

```yaml
services:
  adminer:
    image: adminer
    profiles: ["debug"]
```

```bash
docker compose --profile debug up    # includes adminer
docker compose up                    # excludes it
```

### Scale a service

```bash
docker compose up -d --scale worker=4
```

> **Grounded:** Compose's defining role is local/CI environments. Docker's own [Awesome Compose](https://github.com/docker/awesome-compose) repo is the canonical reference of real multi-service stacks (React+Spring+MySQL, Django+Postgres, etc.). For CI, many projects (e.g. GitLab CI templates) run `docker compose up -d` to stand up dependencies, execute tests with `docker compose run --rm`, then `docker compose down`.

---

## Practical recipes

**Rebuild from scratch (no cache) and restart:**
```bash
docker compose build --no-cache && docker compose up -d
```

**Full reset — tear down containers, networks, AND volumes, then start fresh (⚠️ wipes DB data):**
```bash
docker compose down -v && docker compose up -d
```

**Restart a single service after a code/config change:**
```bash
docker compose up -d --no-deps --build api
```

**Tail logs for one service from now (skip old history):**
```bash
docker compose logs -f --tail=0 api
```

**Run a one-off command in a fresh container (e.g. DB migration):**
```bash
docker compose run --rm api npm run migrate
```

**Open a psql shell in the running db service:**
```bash
docker compose exec db psql -U postgres -d app
```

**Stop everything but keep containers/volumes (fast pause):**
```bash
docker compose stop
```

**Remove this project's stopped containers + default network (keeps named volumes):**
```bash
docker compose down
```

**See resource usage for this project's services:**
```bash
docker compose stats        # or: docker stats $(docker compose ps -q)
```

**Pull newer images and recreate only what changed:**
```bash
docker compose pull && docker compose up -d
```

**Validate / view the fully-merged config (env + overrides resolved):**
```bash
docker compose config
```

**Wipe just one service's data volume (⚠️ destructive):**
```bash
docker compose rm -sv db    # stop, then remove db container + its anonymous volumes
```

---

## Tips

- `down` keeps named volumes; `down -v` deletes them — data loss. Be deliberate.
- Service name = DNS hostname on the Compose network. No need to hardcode IPs.
- Compose is single-host. For multi-host orchestration, that's Kubernetes/Swarm territory.
- Keep secrets out of `compose.yaml`; use `.env` (gitignored) or Docker secrets.
