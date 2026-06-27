# Custom Docker Compose service examples

All files go in the project's `.ddev/` directory and follow the
`docker-compose.<name>.yaml` convention so DDEV merges them. Verify the merged
result with `ddev utility compose-config`.

## Elasticsearch (HTTP, router-exposed, named volume)

`.ddev/docker-compose.elasticsearch.yaml`:

```yaml
services:
  elasticsearch:
    container_name: "ddev-${DDEV_SITENAME}-elasticsearch"
    image: elasticsearch:8.11.0
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    restart: "no"
    expose:
      - "9200"
    environment:
      - VIRTUAL_HOST=${DDEV_HOSTNAME}
      - HTTP_EXPOSE=9200:9200
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - "elasticsearch-data:/usr/share/elasticsearch/data"
      - "./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro"

volumes:
  elasticsearch-data:
    external: true
    name: "${DDEV_SITENAME}-elasticsearch-data"
```

## SQL Server (non-HTTP — requires direct port binding)

A proprietary protocol cannot route through `ddev-router`, so it needs `ports`.
Only one project can use that host port at a time.

`.ddev/docker-compose.sqlsrv.yaml`:

```yaml
services:
  sqlsrv:
    container_name: "ddev-${DDEV_SITENAME}-sqlsrv"
    image: mcr.microsoft.com/mssql/server:2022-latest
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    restart: "no"
    ports:
      - "1433:1433"
    environment:
      - SA_PASSWORD=Password123!
      - ACCEPT_EULA=Y
      - MSSQL_PID=Express
    volumes:
      - "sqlsrv-data:/var/opt/mssql"
      - ".:/mnt/ddev_config"
    platform: linux/amd64   # ARM64 compatibility

volumes:
  sqlsrv-data:
    external: true
    name: "${DDEV_SITENAME}-sqlsrv-data"
```

## Multiple services in one file (Redis + Memcached)

Group related services in a single `.ddev/docker-compose.cache.yaml`:

```yaml
services:
  redis:
    container_name: "ddev-${DDEV_SITENAME}-redis"
    image: redis:7-alpine
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    restart: "no"
    expose:
      - "6379"

  memcached:
    container_name: "ddev-${DDEV_SITENAME}-memcached"
    image: memcached:alpine
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    restart: "no"
    expose:
      - "11211"
```

## Surfacing details in `ddev describe` (`x-ddev`)

```yaml
services:
  rabbitmq:
    container_name: "ddev-${DDEV_SITENAME}-rabbitmq"
    image: rabbitmq:3-management-alpine
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    restart: "no"
    expose:
      - "15672"
    environment:
      - VIRTUAL_HOST=${DDEV_HOSTNAME}
      - HTTP_EXPOSE=15672:15672
      - HTTPS_EXPOSE=15673:15672
      - RABBITMQ_DEFAULT_USER=rabbitmq
      - RABBITMQ_DEFAULT_PASS=rabbitmq
    x-ddev:
      describe-info: |
        User: rabbitmq
        Pass: rabbitmq
      describe-url-port: "extra help here"
```

- `x-ddev.describe-url-port` shows in the URL/PORT column of `ddev describe`.
- `x-ddev.describe-info` shows in the INFO column.

## Optional services via Compose profiles

Services in a named profile are not started on `ddev start`; start them on
demand with `ddev start --profiles=busybox` (or `--profiles='*'` for all):

```yaml
services:
  busybox:
    image: busybox:stable
    command: tail -f /dev/null
    profiles:
      - busybox
    container_name: ddev-${DDEV_SITENAME}-busybox
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
```

## Matching container user to host user

Run the service as the host UID/GID (DDEV provides `DDEV_UID` / `DDEV_GID`):

```yaml
services:
  example:
    container_name: ddev-${DDEV_SITENAME}-example
    image: ${YOUR_DOCKER_IMAGE:-example/example:latest}
    labels:
      com.ddev.approot: ${DDEV_APPROOT}
      com.ddev.site-name: ${DDEV_SITENAME}
    restart: 'no'
    user: "${DDEV_UID}:${DDEV_GID}"
    volumes:
      - .:/mnt/ddev_config
      - ddev-global-cache:/mnt/ddev-global-cache
      - ../:/var/www/html
```

For a more complete setup, create a matching user at build time by passing
`username`, `uid`, `gid` as build args and creating the user in the Dockerfile.

## Trusting DDEV's CA from a third-party service

When a third-party service must consume content from `ddev-webserver` over
HTTPS, by default it does not trust DDEV's `mkcert` CA. Three options:

1. **Use plain HTTP between containers** (simplest) — talk to `http://web`.
2. **Ignore TLS errors** — e.g. `curl --insecure https://web`.
3. **Trust DDEV's CA** — install `mkcert` in the service and mount the global
   cache so the CAROOT is available:

```yaml
# .ddev/docker-compose.example.yaml
services:
  example:
    container_name: ddev-${DDEV_SITENAME}-example
    command: "bash -c 'mkcert -install && original-start-command-from-image'"
    # or, instead of command:
    # post_start:
    #   - command: mkcert -install
    image: ${YOUR_DOCKER_IMAGE:-example/example:latest}-${DDEV_SITENAME}-built
    build:
      context: example
      args:
        YOUR_DOCKER_IMAGE: ${YOUR_DOCKER_IMAGE:-example/example:latest}
    environment:
      - HTTP_EXPOSE=3001:3000
      - HTTPS_EXPOSE=3000:3000
      - VIRTUAL_HOST=$DDEV_HOSTNAME
    external_links:
      - ddev-router:${DDEV_SITENAME}.${DDEV_TLD}   # not needed in DDEV v1.24.10+
    labels:
      com.ddev.approot: ${DDEV_APPROOT}
      com.ddev.site-name: ${DDEV_SITENAME}
    restart: 'no'
    volumes:
      - .:/mnt/ddev_config
      - ddev-global-cache:/mnt/ddev-global-cache   # provides the CAROOT for mkcert
```

Note the `-${DDEV_SITENAME}-built` image suffix on services with a custom
`build`: it lets DDEV operate in offline mode once the base image is pulled.

## Built-in build variants for a service's build stage

You can give a built-in service an inline build stage instead of a separate
Dockerfile:

```yaml
services:
  web:
    build:
      context: .
      dockerfile_inline: |
        FROM ddev-webserver
        RUN apt-get update && apt-get install -y custom-package
        COPY custom-script.sh /usr/local/bin/
```

## Useful DDEV variables in service definitions

`${DDEV_SITENAME}`, `${DDEV_HOSTNAME}` (comma-separated FQDNs), `${DDEV_TLD}`
(`ddev.site`), `${DDEV_APPROOT}`, `${DDEV_DOCROOT}`, `${DDEV_PHP_VERSION}`,
`${DDEV_WEBSERVER_TYPE}`, `${DDEV_DATABASE_FAMILY}`. Project-specific variables
can live in `.ddev/.env` (or `.ddev/.env.<service>`) and be referenced as
`${VAR:-default}`.
