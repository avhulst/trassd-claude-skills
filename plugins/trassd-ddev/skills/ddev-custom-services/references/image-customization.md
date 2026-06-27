# Customizing the web / db images — Dockerfile examples

Add-on Dockerfiles live under `.ddev/web-build/` (web image) or
`.ddev/db-build/` (db image). DDEV inserts their contents into the generated
image build. Insertion order:

- `Dockerfile`, `Dockerfile.*` — **after** DDEV's build steps, alphabetical.
- `pre.Dockerfile`, `pre.Dockerfile.*` — **before** DDEV's steps.
- `prepend.Dockerfile`, `prepend.Dockerfile.*` — at the very **top** (multi-stage).

The `.ddev/*-build` directory is the Docker build context. The final generated
file is `.ddev/.webimageBuild/Dockerfile` — never edit it directly. Force a
clean rebuild with `ddev restart --no-cache` or `ddev utility rebuild`.

Build-time environment variables for the web Dockerfile: `$BASE_IMAGE` (e.g.
`ddev/ddev-webserver:v1.24.0`), `$username`, `$uid`, `$gid`, `$DDEV_PHP_VERSION`,
`$TARGETARCH`, `$TARGETOS`, `$TARGETPLATFORM`. In `prepend.Dockerfile*` only
`$BASE_IMAGE` is auto-available; declare others with `ARG`.

> The image is built with your code NOT mounted and the container not running.
> Build-time operations on `/var/www/html` do nothing. The in-container home dir
> is rebuilt on `ddev restart`, so anything writing to `~` must `USER $username`
> first and `USER root` after.

## PECL extension not in `deb.sury.org` (mcrypt)

`.ddev/web-build/Dockerfile.mcrypt`:

```dockerfile
ENV extension=mcrypt
SHELL ["/bin/bash", "-c"]
RUN (apt-get update || true) && DEBIAN_FRONTEND=noninteractive apt-get install -y -o Dpkg::Options::="--force-confnew" --no-install-recommends --no-install-suggests build-essential php-pear php${DDEV_PHP_VERSION}-dev
RUN apt-get install -y libmcrypt-dev
RUN pecl install ${extension}
RUN echo "extension=${extension}.so" > /etc/php/${DDEV_PHP_VERSION}/mods-available/${extension}.ini && chmod 666 /etc/php/${DDEV_PHP_VERSION}/mods-available/${extension}.ini
RUN phpenmod ${extension}
```

## Overriding the deb.sury.org Xdebug with a PECL build

`.ddev/web-build/Dockerfile.xdebug`:

```dockerfile
ENV extension=xdebug
SHELL ["/bin/bash", "-c"]
RUN phpdismod xdebug
RUN (apt-get update || true) && DEBIAN_FRONTEND=noninteractive apt-get install -y -o Dpkg::Options::="--force-confnew" --no-install-recommends --no-install-suggests build-essential php-pear php${DDEV_PHP_VERSION}-dev
RUN apt-get remove php${DDEV_PHP_VERSION}-xdebug || true
RUN pecl install ${extension}
ADD https://raw.githubusercontent.com/ddev/ddev/main/containers/ddev-php-base/ddev-php-files/etc/php/8.2/mods-available/xdebug.ini /etc/php/${DDEV_PHP_VERSION}/mods-available
RUN chmod 666 /etc/php/${DDEV_PHP_VERSION}/mods-available/xdebug.ini
# ddev xdebug handles enabling, so don't phpenmod here
```

## Building a PHP extension from source for the configured version

Uses `$DDEV_PHP_VERSION` to target the right PHP:

```dockerfile
ENV extension=xhprof
ENV extension_repo=https://github.com/longxinH/xhprof
ENV extension_version=v2.3.8
RUN (apt-get update || true) && DEBIAN_FRONTEND=noninteractive apt-get install -y -o Dpkg::Options::="--force-confnew" --no-install-recommends --no-install-suggests autoconf build-essential libc-dev php-pear php${DDEV_PHP_VERSION}-dev pkg-config zlib1g-dev
RUN mkdir -p /tmp/php-${extension} && cd /tmp/php-${extension} && git clone ${extension_repo} .
WORKDIR /tmp/php-${extension}/extension
RUN git checkout ${extension_version}
RUN phpize
RUN ./configure
RUN make install
RUN echo "extension=${extension}.so" > /etc/php/${DDEV_PHP_VERSION}/mods-available/${extension}.ini
```

## Installing a global CLI tool (phpcs)

`.ddev/web-build/Dockerfile`:

```dockerfile
ENV COMPOSER_HOME=/usr/local/composer
RUN curl -L https://squizlabs.github.io/PHP_CodeSniffer/phpcs.phar -o /usr/local/bin/phpcs && chmod +x /usr/local/bin/phpcs
RUN curl -L https://squizlabs.github.io/PHP_CodeSniffer/phpcbf.phar -o /usr/local/bin/phpcbf && chmod +x /usr/local/bin/phpcbf
RUN chmod 666 /usr/local/bin/CodeSniffer.conf
ENV COMPOSER_HOME=""
```

## Installing into the home directory (playwright)

Switch users so caches land in the right home, then switch back:

```dockerfile
USER $username
RUN npx playwright install
RUN npx playwright install-deps
USER root
```

## Adding an EOL PHP version alongside the main one

Use `pre.Dockerfile.*` so it runs before DDEV's steps. Create
`.ddev/web-build/pre.Dockerfile.php7.4`:

```dockerfile
RUN /usr/local/bin/install_php_extensions.sh "php7.4" "${TARGETARCH}"
```

After `ddev restart`, run `ddev exec php7.4 -v`.

## Multi-stage build

`.ddev/web-build/prepend.Dockerfile` (must re-declare any DDEV build vars it
uses; `$BASE_IMAGE` is global and needed for `FROM`):

```dockerfile
FROM $BASE_IMAGE AS build-stage-go
ARG uid
ARG gid
RUN set -eux; \
    GO_VERSION=$(curl -fsSL "https://go.dev/dl/?mode=json" | jq -r ".[0].version"); \
    AARCH=$(dpkg --print-architecture); \
    wget -q https://go.dev/dl/${GO_VERSION}.linux-${AARCH}.tar.gz -O go.tar.gz; \
    tar -C /usr/local -xzf go.tar.gz; \
    rm go.tar.gz;
```

`.ddev/web-build/Dockerfile`:

```dockerfile
COPY --from=build-stage-go /usr/local/go /usr/local
```

## Debugging the build

1. Prototype interactively with `ddev ssh` (run `sudo killall -USR2 php-fpm`
   after PHP-affecting changes).
2. Move working steps into `.ddev/web-build/Dockerfile`.
3. Run `ddev utility rebuild` for full build output, or
   `export DDEV_VERBOSE=true && ddev start`.
