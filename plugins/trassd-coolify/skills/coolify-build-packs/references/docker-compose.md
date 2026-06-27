# Docker Compose on Coolify — reference

The `docker-compose.y[a]ml` file is the single source of truth. Define env vars,
storage, and healthcheck behavior in the file itself.

## Exposing services to the internet

Three options:

1. **Domain** — assign a domain to a service in Coolify. If the service listens on
   port 80, a domain is enough for the proxy to route to it. Otherwise add the
   container port: enter `http://example.com:3000` for a service on port 3000. The
   port only tells Coolify where to send traffic *inside* the container; the proxy
   still serves it on the normal port (80/443).
2. **Service port mapping** — add `ports:` to publish on the host. This bypasses
   the proxy and exposes the port directly on the server. Optionally bind to an
   interface, e.g. `"127.0.0.1:3000:3000"` to keep it on localhost only.

   ```yaml
   services:
     backend:
       image: your-backend:latest
       ports:
         - "3000:3000"          # reachable on server:3000, outside proxy control
   ```

3. **Internal only** — no domain and no `ports:` keeps the service private. Other
   services reach it by name, e.g. `http://auth:1234`.

## Networking

Coolify auto-creates one isolated bridge network per stack (named after the
resource UUID) and attaches Traefik to it. Services talk to each other by service
name.

**Never define custom `networks:`.** Doing so places containers on two networks at
once; Traefik non-deterministically picks an IP and may route to the unreachable
one, causing intermittent hangs or 504 Gateway Timeout. Remove all `networks:`
blocks — the auto-created network already provides inter-service communication.

**Cross-stack:** to connect to a service in a different stack, enable *Connect to
Predefined Network* on the Service Stack page, then reference the other service by
its full name `name-<uuid>` (e.g. `postgres-<uuid>`). Note that enabling this makes
the internal Docker DNS behave differently, hence the full-name requirement.

## Environment variables

Coolify detects env vars in the compose file and surfaces editable ones in the UI.

```yaml
services:
  myservice:
    environment:
      - SOME_HARDCODED_VALUE=hello                       # passed to container, NOT shown in UI
      - SOME_VARIABLE=${SOME_VARIABLE_IN_COOLIFY_UI}     # uninitialized, editable in UI
      - SOME_DEFAULT=${OTHER_NAME:-hello}                # default "hello", editable in UI
```

### Required variables

Mark with `:?`. Deployment is blocked until set; empty ones get a red border.

```yaml
environment:
  - DATABASE_URL=${DATABASE_URL:?}        # required, no default
  - PORT=${PORT:?3000}                    # required, prefilled default, editable
  - DEBUG=${DEBUG:-false}                 # optional, standard behavior
```

Validation happens before container creation, preventing partial deployments.

### Shared variables

Coolify does not auto-detect shared variables. Create the shared variable, define
the placeholder in compose, then in the app's Environment Variables set the name to
the compose placeholder with value `{{environment.SOME_SHARED_VARIABLE}}`.

## Magic environment variables

Syntax: `SERVICE_<TYPE>_<IDENTIFIER>`. Coolify generates a value, reusable across
services (same identifier → same value). Generated values are editable in the UI
except FQDN and URL.

| Type | Generated value |
| --- | --- |
| `URL` | URL based on your wildcard domain (supports paths/ports) |
| `FQDN` | FQDN based on the generated URL |
| `USER` | Random 16-char string |
| `PASSWORD` / `PASSWORD_64` | Random password without symbols (16 / 64 chars) |
| `PASSWORDWITHSYMBOLS` / `_64` | Random password with symbols (default / 64 chars) |
| `BASE64` / `BASE64_32/64/128` | Random string (not Base64-encoded), 32/64/128 chars |
| `REALBASE64` / `_32/64/128` | Base64-encoded random string, 32/64/128 chars |
| `HEX_32/64/128` | Hexadecimal random string, 32/64/128 chars |

Identifiers with underscores cannot use ports — use hyphens:
`SERVICE_URL_APPWRITE-SERVICE_3000` (not `..._APPWRITE_SERVICE_3000`).

```yaml
services:
  appwrite:
    environment:
      - SERVICE_URL_APPWRITE                              # http://appwrite-<uuid>.example.com
      - SERVICE_URL_APPWRITE_3000                         # proxied to port 3000
      - DOMAIN_NAME=${SERVICE_FQDN_APPWRITE}              # full FQDN
      - SECRET=${SERVICE_PASSWORD_64_APPWRITE}            # generated 64-char password
      - ENCRYPTION_KEY=${SERVICE_REALBASE64_64_APPWRITE}  # 64-char Base64 string
  not-appwrite:
    environment:
      - APPWRITE_PASSWORD=${SERVICE_PASSWORD_APPWRITE}    # reuses appwrite's password
```

Magic env vars in Git-sourced compose files require Coolify v4.0.0-beta.411+.

## Storage

Coolify adds compose extensions to manage storage:

```yaml
services:
  filebrowser:
    image: filebrowser/filebrowser:latest
    volumes:
      - type: bind
        source: ./srv
        target: /srv
        is_directory: true        # tells Coolify to create the directory
```

Create a file with content (and inject an env value):

```yaml
    volumes:
      - type: bind
        source: ./srv/99-roles.sql
        target: /docker-entrypoint-initdb.d/init-scripts/99-roles.sql
        content: |                 # tells Coolify to create the file
          \set pgpass `echo "$POSTGRES_PASSWORD"`
          ALTER USER authenticator WITH PASSWORD :'pgpass';
```

Alternatively use the top-level `configs:` element with a `content:` field.

## Exclude from healthchecks

For run-once services (e.g. a migration that exits), exclude them so they don't
fail the stack's overall healthcheck:

```yaml
services:
  migrate:
    exclude_from_hc: true
```

## Labels

Coolify adds these labels if unset:

```yaml
labels:
  - coolify.managed=true
  - coolify.applicationId=5
  - coolify.type=application
```

To route through Coolify's Traefik proxy, add:

```yaml
labels:
  - traefik.enable=true
  - "traefik.http.routers.<unique_router_name>.rule=Host(`example.com`) && PathPrefix(`/`)"
  - traefik.http.routers.<unique_router_name>.entryPoints=http
```

## Raw Compose Deployment

Deploys the compose file directly without most of Coolify's magic. Intended for
advanced users familiar with Docker Compose.
