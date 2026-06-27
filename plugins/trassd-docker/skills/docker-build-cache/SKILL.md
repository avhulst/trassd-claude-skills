---
name: docker-build-cache
description: Optimize the Docker build cache for fast, deterministic builds — how layers are cached and invalidated, ordering instructions for cache hits, cache mounts (RUN --mount=type=cache), bind mounts, and external cache backends (registry, GitHub Actions). Use when builds are slow, when a small change rebuilds too much, or to improve cache reuse in CI.
---

# Docker build cache

Each instruction in a Dockerfile produces a layer, stacked on top of the layers
before it. A layer is reused from the cache when the instruction and the files
it depends on haven't changed since it was last built. Reusing a layer skips
rebuilding it, which is what makes repeat builds fast.

## How invalidation works

The builder steps through the Dockerfile in order, checking each instruction
against the cached layers. It starts by checking whether the base image is
cached, then compares each subsequent instruction.

- **Once a layer is invalidated, every layer after it is rebuilt too** — even
  if those later instructions would produce identical output, they still
  re-run. So a change low in the file is cheap; a change high in the file is
  expensive.
- For most instructions, comparing the instruction string against the cached
  layer is enough. `RUN apt-get -y update` matches purely on the command
  string — the builder does not inspect files changed inside the container to
  decide a hit.
- For `ADD`, `COPY`, and `RUN --mount=type=bind`, the builder computes a cache
  checksum from **file metadata**. If that metadata changed for any involved
  file, the cache is invalidated. The file's modification time (`mtime`) is
  **not** part of the checksum — touching a file without changing its contents
  does not bust the cache.

## Order your layers

Put expensive, rarely-changing steps near the top; put frequently-changing
steps (like copying source code) near the bottom. The classic fix is splitting
a single `COPY . .` so dependencies install before source is copied:

```dockerfile
# syntax=docker/dockerfile:1
FROM node
WORKDIR /app
COPY package.json yarn.lock .    # rarely changes
RUN npm install                  # cached unless deps change
COPY . .                         # changes often
RUN npm build
```

With `COPY . .` first, editing any source file re-runs `npm install` every
time, even when dependencies are unchanged. Splitting the copy keeps the
install layer cached across source edits.

## Keep the build context small

The context is the set of files sent to the builder. A smaller context means
less data transferred and fewer chances for cache invalidation. Add a
`.dockerignore` at the context root to exclude files you never need in the
build:

```plaintext
node_modules
tmp*
```

Ignore rules apply to the whole context including subdirectories.

## Bind mounts: avoid baking unneeded files into the cache

If `COPY` adds files only used to produce an artifact, BuildKit puts all of
them in the cache even when they don't end up in the final image. A bind mount
makes the files available for the duration of one `RUN` only, without adding a
layer:

```dockerfile
FROM golang:latest
WORKDIR /build
RUN --mount=type=bind,target=. go build -o /app/hello
```

Notes:
- Bind mounts are read-only by default; use the `rw` option to write, but
  writes are discarded after the `RUN` — they do not persist in the image or
  cache.
- Mounted files are not in the final image. Use `COPY`/`ADD` for files you need
  to keep.
- Write build output **outside** the mount target (here `/app/hello`), since
  the mount is read-only.

## Cache mounts: persistent package caches

A regular layer is all-or-nothing. A cache mount gives a `RUN` a persistent,
cumulative directory that survives across builds — so even when the layer
rebuilds, the package manager only fetches new or changed packages:

```dockerfile
FROM node:latest
WORKDIR /app
RUN --mount=type=cache,target=/root/.npm npm install
```

Point the mount at the package manager's cache directory. Examples:

```dockerfile
# pip
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

# Go
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /app/hello

# apt — needs exclusive access, so lock the cache
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt update && apt-get --no-install-recommends install -y gcc
```

Check your package manager's docs for the right target path and options. apt
needs `sharing=locked` so parallel builds sharing the mount wait for each other
instead of racing on the same files.

## External cache for CI

The default cache is internal to the builder (BuildKit instance) you're using,
and is not shared when you switch builders. CI builders are often ephemeral, so
an external cache — stored at a remote location and shared across builds and
environments — can drastically cut build time and cost.

Use `docker buildx build` with:

- `--cache-from` — remote caches the build may pull from.
- `--cache-to` — exports the build cache to the given location.

GitHub Actions example pushing cache to a registry image:

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    push: true
    tags: user/app:latest
    cache-from: type=registry,ref=user/app:buildcache
    cache-to: type=registry,ref=user/app:buildcache,mode=max
```

`mode=max` exports cache for all stages, not just the final one. The same cache
works locally:

```bash
docker buildx build --cache-from type=registry,ref=user/app:buildcache .
```

## Forcing a rebuild

`RUN` cache is not invalidated automatically between builds — `RUN apk add
curl` keeps the same package version on a rebuild a week later. To force
re-execution:

- Change a layer before it.
- Clear the cache first with `docker builder prune`.
- Build with `--no-cache`, or target one stage with
  `--no-cache-filter <stage>`.

```bash
docker build --no-cache-filter install .
```

## Reproducible builds and SOURCE_DATE_EPOCH

`WORKDIR` respects the `SOURCE_DATE_EPOCH` build argument when deciding cache
validity, and changing it invalidates `WORKDIR` and everything after. Setting
it to a dynamic value like a Git commit timestamp breaks the cache on every
commit (expected when tracking provenance). For reproducible builds without
constant invalidation, pin it:

```bash
docker build --build-arg SOURCE_DATE_EPOCH=0 .
```
