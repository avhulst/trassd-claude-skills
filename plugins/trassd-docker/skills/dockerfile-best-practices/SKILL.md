---
name: dockerfile-best-practices
description: Write clean, efficient, reproducible Dockerfiles — instruction ordering for layer caching, minimal layers, COPY vs ADD, pinned base images, a non-root USER, ENTRYPOINT vs CMD, and a tight build context via .dockerignore. Use when creating or editing a Dockerfile.
---

# Dockerfile best practices

Guidance for writing clean, reliable, reproducible Dockerfiles. Apply when
authoring or editing a `Dockerfile`.

## Start with the syntax directive

Make the first line a parser directive so BuildKit uses the current Dockerfile
frontend. `docker/dockerfile:1` always points to the latest v1 release and is
checked for updates before each build. The directive must come before any other
comment, whitespace, or instruction.

```dockerfile
# syntax=docker/dockerfile:1
```

## Choose a small, trusted base image

The base is the `FROM` image your image extends. Pick a minimal image from a
trusted source — prefer Docker Official Images, Verified Publisher, or
Docker-Sponsored Open Source images (look for the badge on Docker Hub). A
smaller base downloads faster, shrinks the final image, and reduces the number
of vulnerabilities pulled in through dependencies. Docker recommends Alpine
(under 6 MB) as a tightly controlled yet full Linux distribution.

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.21
```

Use `FROM scratch` only for minimal images that contain just what an
application needs (for example, a statically linked binary with no runtime
dependencies). Note that programs often depend on language runtimes,
dynamically linked C libraries, or CA certificates that `scratch` does not
provide.

Don't install unnecessary packages. A database image doesn't need a text
editor. Fewer packages means reduced complexity, dependencies, file size, and
build time.

## Pin base image versions

Image tags are mutable: a publisher can repoint `3.21` to a newer patch over
time, so the same tag isn't guaranteed to resolve to the same image on every
build. That can introduce breaking changes and leaves no audit trail. To fully
secure supply-chain integrity, pin to a digest — your build then always uses
the exact same image even if the publisher replaces the tag.

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.21@sha256:a8560b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c
```

The trade-off: pinning a digest is more tedious to update and opts you out of
automatic security fixes, so you must update the digest deliberately.

## Rebuild often to stay current

Images are immutable snapshots — they include whatever base, libraries, and
software existed at build time. Rebuild regularly with updated dependencies.

- `docker build --pull` forces a check for and download of a newer base image,
  even if a version is cached locally.
- `docker build --no-cache` disables the build cache and rebuilds every layer,
  picking up the latest package-manager versions (`apt-get`, `npm`). It does
  not pull a fresh base image.
- Combine them to get both: `docker build --pull --no-cache -t my-image .`

## Order instructions for the build cache

Docker executes instructions in order and reuses a cached layer when an
instruction is unchanged. Once a layer's cache is invalidated, every layer
after it is rebuilt. So order from least- to most-frequently changing: install
dependencies before copying application source, so editing source doesn't bust
the dependency-install layer.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:latest
WORKDIR /src
COPY package.json package-lock.json .
RUN npm ci
COPY index.ts src .
```

## Keep RUN instructions clean and combined

Split long `RUN` statements across multiple lines with backslashes for
readability, and chain related commands with `&&`. Sort multi-line arguments
alphanumerically to avoid duplicate packages and make diffs easier to review.

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    package-bar \
    package-baz \
    package-foo \
    && rm -rf /var/lib/apt/lists/*
```

For `apt-get`, always combine `apt-get update` with `apt-get install` in the
same `RUN`. Splitting them lets Docker reuse a stale `update` layer when only
the `install` line changes, so you can get outdated packages — this combined
form is known as cache busting. Cleaning `/var/lib/apt/lists` in the same layer
keeps the apt cache out of the image. You can also use a here-document instead
of chaining with `&&`:

```dockerfile
RUN <<EOF
apt-get update
apt-get install -y --no-install-recommends package-foo
rm -rf /var/lib/apt/lists/*
EOF
```

When piping commands, the shell only checks the exit code of the last command
in the pipe, so a failed earlier stage can succeed silently. Prepend
`set -o pipefail &&` to fail the build on any pipe error.

## COPY vs ADD

`COPY` and `ADD` are similar, but use the narrower one by default:

- **`COPY`** for copying files from the build context or from another stage in
  a multi-stage build. This is the common case.
- **`ADD`** when you need to download a remote artifact (HTTPS or Git URL) as
  part of the build. `ADD` gives a more precise build cache than manual
  `wget`/`tar`, and supports checksum validation of remote resources.

To temporarily add a build-context file just for one `RUN` (e.g. a
`requirements.txt` for `pip install`), prefer a bind mount over `COPY` — it's
more efficient and the file doesn't persist in the final image. Use `COPY` when
the file must end up in the image.

```dockerfile
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install --requirement /tmp/requirements.txt
```

## Tighten the build context with .dockerignore

The build context is the set of files the build can access. A `.dockerignore`
file in the context root excludes files before they're sent to the builder,
which improves build speed (especially with a remote builder) and avoids
shipping unwanted files. Patterns are newline-separated globs, similar to
`.gitignore`.

```text
# .dockerignore
node_modules
*.md
!README.md
```

Notes:

- `**` matches any number of directories: `**/*.go` excludes all `.go` files
  anywhere in the context.
- A `!` prefix negates a match (re-includes); the **last** matching line wins,
  so order matters.
- With multiple Dockerfiles, a `<dockerfile-name>.dockerignore` placed next to
  the Dockerfile takes precedence over the root `.dockerignore`.

## Run as a non-root USER

If a service can run without privileges, create a user and switch to it with
`USER`. Assign an explicit UID/GID if it's critical, since auto-assigned IDs
are non-deterministic across rebuilds. To reduce layers and complexity, avoid
switching `USER` back and forth. Avoid `sudo` (unpredictable TTY and
signal-forwarding behavior); use `gosu` if you need root-then-drop behavior.

```dockerfile
RUN groupadd -r appuser && useradd --no-log-init -r -g appuser appuser
USER appuser
```

## ENTRYPOINT vs CMD

- **`CMD`** sets the default command (and/or default arguments). Prefer the
  exec form `CMD ["executable", "param1", "param2"]` for service images. Only
  the last `CMD` in a Dockerfile is respected.
- **`ENTRYPOINT`** sets the image's main command so the image runs as though it
  were that command; `CMD` then supplies the default flags.

```dockerfile
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```

Prefer the exec form (JSON array) over the shell form — they differ in how
signals like `SIGTERM` are trapped. When using an entrypoint helper script, use
the shell `exec` builtin so the application becomes PID 1 and receives Unix
signals:

```dockerfile
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["postgres"]
```

## Other instruction tips

- **WORKDIR**: always use absolute paths; prefer `WORKDIR` over
  `RUN cd … && do-something`.
- **ENV**: useful for `PATH` updates and pinning version numbers. Each `ENV`
  creates a layer and the value persists even if unset later — to truly remove
  a secret-ish variable, set, use, and unset it within a single `RUN`.
- **EXPOSE**: document the port your service listens on (e.g. `EXPOSE 8000`).
  It's not required, but helps tools and teammates.
- **LABEL**: add key-value metadata for project, licensing, or automation.
  Quote or escape values with spaces.

## Design for ephemeral, single-concern containers

Containers should be ephemeral — able to be stopped, destroyed, and rebuilt
with minimal setup. Give each container a single concern (web app, database,
in-memory cache as separate images) so you can scale and reuse them; connect
them with Docker networks. Use multi-stage builds to keep build tooling out of
the runtime image — see the `docker-multi-stage-builds` skill.
