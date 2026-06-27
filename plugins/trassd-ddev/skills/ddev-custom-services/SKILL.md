---
name: ddev-custom-services
description: >-
  Add extra services and Docker customizations to a DDEV project — additional
  services, custom docker-compose.*.yaml files, custom Docker services,
  additional hostnames, and image customization. Triggers when adding a service
  (Redis, Elasticsearch, …) or a docker-compose.*.yaml under .ddev, when adding
  hostnames/FQDNs to config.yaml, or when extending the web/db images with extra
  packages, PHP extensions, or a custom Dockerfile.
---

# DDEV custom services & Docker customizations

DDEV runs a project as multiple containers via a private copy of Docker Compose.
You extend it by dropping extra Compose files and build files into `.ddev/`.
Always prefer an **add-on** to a hand-rolled service when one exists; reach for a
custom `docker-compose.*.yaml` only when no add-on fits.

## Add-ons vs. hand-rolled services

- **Use an add-on** when a standard, tested service already exists (Redis,
  Elasticsearch, Solr, …). Install with `ddev add-on get <name>`; it provides
  automatic config and updates. Browse the registry at <https://addons.ddev.com/>.
- **Hand-roll a custom service** when you need something specialized, deep
  customization, prototyping, or tight project-specific integration. A stable,
  reusable custom service should eventually be promoted to an add-on (use the
  [ddev-addon-template](https://github.com/ddev/ddev-addon-template)).

## Adding a custom service

DDEV merges **every** file in `.ddev/` matching the `docker-compose.*.yaml`
naming convention into the full Compose config. Create
`.ddev/docker-compose.<service>.yaml` — never edit
`.ddev/.ddev-docker-compose-base.yaml` or `.ddev/.ddev-docker-compose-full.yaml`
(both are regenerated and overwritten on every start).

Required conventions for each service so DDEV treats it like a built-in service:

- **Container name** must be `ddev-${DDEV_SITENAME}-<servicename>` — the Traefik
  router config is generated from this pattern.
- **Labels** — add both (auto-added since DDEV v1.25.2+, but include them for
  older versions and clarity):

  ```yaml
  labels:
    com.ddev.site-name: ${DDEV_SITENAME}
    com.ddev.approot: ${DDEV_APPROOT}
  ```

- **Restart policy** `restart: "no"`.
- **Config volume** mount `.:/mnt/ddev_config` so the service sees DDEV config.

Minimal HTTP service (`.ddev/docker-compose.myservice.yaml`):

```yaml
services:
  myservice:
    container_name: "ddev-${DDEV_SITENAME}-myservice"
    image: nginx:alpine
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    restart: "no"
    expose:
      - "8080"
    environment:
      - VIRTUAL_HOST=${DDEV_HOSTNAME}
      - HTTP_EXPOSE=8080:8080
      - HTTPS_EXPOSE=8081:8080
    volumes:
      - ".:/mnt/ddev_config"
```

After editing, verify the merged result with `ddev utility compose-config` (or
inspect `.ddev/.ddev-docker-compose-full.yaml`), then `ddev restart`.

## Exposing a service

**Prefer router-based exposure over `ports`.** Direct `ports` binding prevents
running multiple projects with the same service at once, so use it **only** for
non-HTTP protocols that cannot route through DDEV (e.g. a raw DB port).

For HTTP services, declare the **internal** port under `expose:` and route it
through `ddev-router` via environment variables:

- `VIRTUAL_HOST=$DDEV_HOSTNAME` — the project FQDN(s). May be a subdomain
  (`mysubdomain.$DDEV_HOSTNAME`) or an arbitrary host (`extra.ddev.site`).
- `HTTP_EXPOSE=hostPort:containerPort` — comma-separate to expose several ports.
- `HTTPS_EXPOSE=exposedPort:containerPort` — serves
  `https://<project>.ddev.site:exposedPort`; comma-separate for multiple, e.g.
  `HTTPS_EXPOSE=9998:80,9999:81`.

Non-HTTP service needing direct binding:

```yaml
services:
  special-service:
    ports:
      - "9999:9999"   # only when HTTP routing is impossible
```

## Customizing or adding to existing services

You can target a built-in service (`web`, `db`) from your own
`docker-compose.*.yaml` to add env vars, volumes, or a build stage:

```yaml
services:
  web:
    environment:
      - CUSTOM_ENV_VAR=value
    volumes:
      - ./custom-config:/etc/custom-config:ro
```

Useful patterns: surface service details in `ddev describe` with the `x-ddev`
extension field; run resource-heavy services on demand via Compose `profiles`
(started with `ddev start --profiles=<name>`); align container user with the host
via `user: "${DDEV_UID}:${DDEV_GID}"`. See
[references/compose-services.md](references/compose-services.md) for full
examples (Elasticsearch, SQL Server, Redis/Memcached, `x-ddev`, profiles, user
matching, and the mkcert-trust pattern for cross-container HTTPS).

Interact with a custom service via the `--service`/`-s` flag:
`ddev logs -s myservice`, `ddev exec -s myservice bash`, `ddev ssh -s myservice`.

## Additional hostnames & FQDNs

Add extra names in `.ddev/config.yaml` (or via `ddev config`):

```yaml
name: mysite
additional_hostnames:
  - extraname        # -> extraname.ddev.site
  - fr.mysite        # -> fr.mysite.ddev.site
  - "*.lotsofnames"  # wildcard (needs internet + ddev.site TLD + DNS)
```

Equivalent: `ddev config --additional-hostnames extraname,fr.mysite,*.lotsofnames`.

`additional_fqdns` registers names **outside** `.ddev.site` via the hosts file —
use with care:

```yaml
additional_fqdns:
  - example.com
  - somesite.example.com
```

- If the FQDN resolves on the internet, set `use_dns_when_possible: false`.
- Never reuse the same hostname/FQDN across two projects — it breaks
  `ddev-router` (shows unhealthy in `ddev list`).
- Never override a real domain you still need to reach (e.g. `www.google.com`).

## Customizing the web / db images

Two ways to extend the `web` and `db` images:

1. **Extra Debian packages** in `.ddev/config.yaml` — simplest:

   ```yaml
   webimage_extra_packages: ['php${DDEV_PHP_VERSION}-tidy', locales-all]
   dbimage_extra_packages: [netcat, telnet, sudo]
   ```

   For PHP extensions available from `deb.sury.org`, test with
   `ddev exec '(sudo apt-get update || true) && sudo apt-get install php${DDEV_PHP_VERSION}-<ext>'`,
   then add `php${DDEV_PHP_VERSION}-<ext>` to `webimage_extra_packages`.

2. **Add-on Dockerfiles** under `.ddev/web-build/` or `.ddev/db-build/` for
   anything more complex (PECL extensions, global CLI tools, EOL PHP versions,
   multi-stage builds). DDEV inserts these into the generated image build:

   - `Dockerfile` and `Dockerfile.*` — appended **after** DDEV's build steps,
     in alphabetical order.
   - `pre.Dockerfile` / `pre.Dockerfile.*` — inserted **before** DDEV's steps
     (proxy/SSL setup, EOL PHP).
   - `prepend.Dockerfile` / `prepend.Dockerfile.*` — inserted at the very top,
     for multi-stage builds. Only `$BASE_IMAGE` is auto-available here; declare
     any other build var with `ARG`.

   Minimal example (`.ddev/web-build/Dockerfile`):

   ```dockerfile
   RUN npm install -g gatsby-cli
   ```

   The `.ddev/*-build` directory is the Docker build context, so `ADD file.txt /`
   works for files placed there. Build-time vars include `$BASE_IMAGE`,
   `$DDEV_PHP_VERSION`, `$username`, `$uid`, `$gid`, `$TARGETARCH`, `$TARGETOS`,
   `$TARGETPLATFORM`. The generated Dockerfile lands at
   `.ddev/.webimageBuild/Dockerfile` (never edit it). Force a clean rebuild with
   `ddev restart --no-cache` or `ddev utility rebuild` (the latter shows full
   build output for debugging).

   **The image builds with your code NOT mounted and the container not running** —
   anything operating on `/var/www/html` at build time does nothing. Things that
   write to the home directory (e.g. `~/.cache`) need `USER $username` before and
   `USER root` after, since the home dir is rebuilt on `ddev restart`.

   See [references/image-customization.md](references/image-customization.md) for
   PECL extension Dockerfiles (mcrypt, xdebug override, building from source),
   global tool installs, EOL PHP, and a multi-stage build.
