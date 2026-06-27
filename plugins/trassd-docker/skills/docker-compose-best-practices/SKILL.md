---
name: docker-compose-best-practices
description: Authoring maintainable Compose files using the services/networks/volumes model, depends_on with healthcheck conditions for startup order, profiles, restart policies, and production-readiness. Triggers when creating or editing a compose.yaml / compose.yml / docker-compose.yml.
---

# Docker Compose best practices

Guidance for writing clear, maintainable Compose files based on the Compose
Specification and Compose how-to docs.

## File naming and structure

- Name the file `compose.yaml` (preferred) or `compose.yml`, placed in the
  working directory. `docker-compose.yaml` / `docker-compose.yml` are supported
  only for backwards compatibility; if both exist, Compose prefers the canonical
  `compose.yaml`.
- The Compose Specification is the current, recommended format. There is no
  `version:` top-level field to maintain — legacy 2.x / 3.x versions were merged
  into the Specification.
- Set a top-level `name:` to control the project name used to group and isolate
  resources. The same `compose.yaml` can be deployed more than once on the same
  infrastructure by passing a distinct project name.

## The application model

A Compose file defines four kinds of top-level elements:

- **services** — the computing components. A service runs the same container
  image and configuration one or more times.
- **networks** — how services communicate. The mere presence of a network
  entry is enough to define it (`front-tier: {}`).
- **volumes** — persistent data shared by services, declared as high-level
  filesystem mounts.
- **configs** / **secrets** — runtime/platform-dependent configuration and
  sensitive data, mounted into containers as files.

With volumes, configs, and secrets you can declare them simply at the top level
and add platform-specific detail at the service level.

```yaml
services:
  frontend:
    image: example/webapp
    ports:
      - "443:8043"
    networks:
      - front-tier
      - back-tier
  backend:
    image: example/database
    volumes:
      - db-data:/etc/data
    networks:
      - back-tier

volumes:
  db-data:

networks:
  front-tier: {}
  back-tier: {}
```

Isolate traffic with multiple networks: connect each service only to the
networks it needs (for example, a back-tier network shared by frontend and
backend, plus a front-tier network only the frontend joins).

## Building images

- `build:` as a string is a path to the build context, which must contain a
  `Dockerfile`. Relative paths resolve from the directory containing the Compose
  file.
- Prefer relative build paths. Absolute paths make the Compose file non-portable
  and Compose emits a warning.
- Use the long form to set an alternate Dockerfile or build args:

  ```yaml
  services:
    backend:
      image: example/database
      build:
        context: backend
        dockerfile: backend.Dockerfile
        args:
          GIT_COMMIT: cdc3b19
  ```

- Set both `build:` and `image:` so built images can be tagged and pushed.
  Compose skips pushing any service that has `build:` but no `image:`, and warns.
- When both `build:` and `image:` are present, the `pull_policy` attribute
  governs behavior. With no `pull_policy`, Compose tries to pull first and builds
  from source only if the image isn't found.
- Use `target:` to build a specific stage of a multi-stage Dockerfile (for
  example `target: prod`).

## Startup and shutdown order

Compose starts and stops containers in dependency order, derived from
`depends_on`, `links`, `volumes_from`, and `network_mode: "service:..."`.

On startup Compose only waits until a dependency is *running*, not *ready*. A
service that connects to a database may start before the database can accept
connections. Control readiness with `depends_on` long form plus a `condition`:

- `service_started` — dependency has started.
- `service_healthy` — dependency reports healthy via its `healthcheck` before
  the dependent service starts.
- `service_completed_successfully` — dependency ran to successful completion
  first (useful for migrations/init jobs).

```yaml
services:
  web:
    build: .
    depends_on:
      db:
        condition: service_healthy
        restart: true
      redis:
        condition: service_started
  redis:
    image: redis
  db:
    image: postgres:18
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s
```

- Pair `service_healthy` with a real `healthcheck` on the dependency — without
  one, the condition cannot be satisfied.
- `restart: true` restarts the dependent service when the dependency is updated
  or restarted by an explicit Compose operation, so it re-establishes
  connections.
- Use `$$` to escape a literal `$` so Compose does not interpolate it (the
  `pg_isready` example passes `${POSTGRES_USER}` through to the shell).
- Compose removes services in reverse dependency order, so dependents stop
  before their dependencies.

## Profiles

Use `profiles` to keep optional services (debug tools, one-off utilities) out of
the default `docker compose up`.

```yaml
services:
  backend:
    image: backend
  db:
    image: mysql
  phpmyadmin:
    image: phpmyadmin
    depends_on: [db]
    profiles: [debug]
```

- A service with no `profiles` is always enabled. Keep your application's **core
  services unprofiled** so they always start.
- Enable profiles with `--profile <name>` (repeatable) or the
  `COMPOSE_PROFILES` env var (comma-separated). `--profile "*"` enables all.
- Explicitly targeting a profiled service (for example `docker compose run
  db-migrations`) starts it without enabling its profile, along with its
  `depends_on` dependencies. Other services sharing that profile do not start
  unless also targeted or the profile is enabled.
- Profile names must match `[a-zA-Z0-9][a-zA-Z0-9_.-]+`.

## Restart policies and production-readiness

For production, adapt the development Compose file rather than maintaining a
separate, divergent one — typically by layering a second file:

- Specify a restart policy such as `restart: always` to avoid downtime.
- Remove volume bindings for application code so code stays inside the container
  and can't be changed from outside.
- Bind to different host ports as needed.
- Set environment variables differently (for example, reduce log verbosity or
  point to external services).
- Add production-only services such as a log aggregator.

Keep production-specific changes in a separate overlay file (for example
`compose.production.yaml`) containing only the differences, and apply it over
the base file:

```bash
docker compose -f compose.yaml -f compose.production.yaml up -d
```

When redeploying after code changes, rebuild and recreate only the affected
service; `--no-deps` avoids recreating its dependencies:

```bash
docker compose build web
docker compose up --no-deps -d web
```

To deploy to a remote host, set `DOCKER_HOST`, `DOCKER_TLS_VERIFY`, and
`DOCKER_CERT_PATH`; the normal `docker compose` commands then work unchanged.

## Keeping files maintainable

- Factor out shared definitions with fragments and extensions; reuse other files
  with `include`.
- Merge multiple Compose files (`-f` order matters): scalars and maps are
  overridden by the highest-order file, lists are appended. Relative paths
  resolve against the first file's parent folder.
