---
name: docker-build-ci
description: Building and pushing Docker images in CI with GitHub Actions — the official Docker actions, build-push-action, cache import/export, multi-platform builds, automatic tags and labels, build secrets, and testing an image before pushing. Use when setting up or reviewing a Docker build workflow under .github/workflows.
---

# Docker image builds in CI (GitHub Actions)

Docker maintains official GitHub Actions for building, annotating, and pushing
images with BuildKit. Use a Docker container as a reproducible, isolated build
environment, and these actions to wire it into a workflow.

## Official Docker actions

- `docker/build-push-action` — build and push images with BuildKit.
- `docker/setup-buildx-action` — create and boot a BuildKit builder.
- `docker/setup-qemu-action` — install QEMU static binaries for multi-platform
  builds via emulation.
- `docker/login-action` — sign in to a registry.
- `docker/metadata-action` — extract metadata from the Git ref and GitHub
  events to generate tags, labels, and annotations.
- `docker/bake-action` — high-level builds with Bake.

Pin actions to a released version (`uses: docker/build-push-action@vX.Y.Z`).

## Baseline build-and-push workflow

```yaml
name: ci

on:
  push:

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: user/app:latest
```

Store registry credentials as repository/organization secrets (or vars for the
non-secret username); never hardcode them.

## Automatic tags and labels

Let `docker/metadata-action` generate tags and OCI labels from Git metadata and
events instead of maintaining them by hand. Reference its outputs from the build
step:

```yaml
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            name/app
            ghcr.io/username/app
          tags: |
            type=schedule
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

Gate pushes (and logins) on `github.event_name != 'pull_request'` so PR builds
validate without publishing. Push multiple registries by listing multiple
`images` and adding a login step per registry — for GHCR, log in to `ghcr.io`
using `${{ github.repository_owner }}` and `${{ secrets.GITHUB_TOKEN }}`.

## Multi-platform builds

Use the `platforms` input. Add `setup-qemu-action` when relying on emulation
for non-native platforms:

```yaml
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          platforms: linux/amd64,linux/arm64
          push: true
          tags: user/app:latest
```

GitHub runners support building and pushing multi-platform images, but not
loading them locally. To `load: true` a multi-platform image into the runner's
image store, enable the containerd image store via `docker/setup-docker-action`:

```yaml
      - name: Set up Docker
        uses: docker/setup-docker-action@v4
        with:
          daemon-config: |
            {
              "features": {
                "containerd-snapshotter": true
              }
            }
```

Building all platforms on one runner can be slow. To split platforms across
runners without a custom matrix and merge job, use the Docker GitHub Builder
reusable workflows, which compute the per-platform matrix, run each platform on
its own runner, and create the final manifest.

## Caching

`build-push-action` supports BuildKit cache import/export via `cache-from` /
`cache-to`. Choose a backend:

- GitHub Actions cache (`type=gha`) — natural fit for GitHub-hosted runners.
  Use `mode=max` to cache all layers including intermediate stages.

  ```yaml
          cache-from: type=gha
          cache-to: type=gha,mode=max
  ```

  Only GitHub Cache service API v2 is supported (the v1 API was shut down April
  15, 2025). If you hit a "legacy service is shutting down" error on self-hosted
  runners, upgrade Buildx (>= v0.21.0) and BuildKit (>= v0.20.0), or install the
  latest with `setup-buildx-action` (`with: version: latest`).

- Registry cache (`type=registry`) — stores cache in a registry image; supports
  `mode=max`.

  ```yaml
          cache-from: type=registry,ref=user/app:buildcache
          cache-to: type=registry,ref=user/app:buildcache,mode=max
  ```

- Inline cache (`type=inline`) — embeds cache in the image; only supports
  `min` mode. For `max`, use the registry exporter instead.

  ```yaml
          cache-from: type=registry,ref=user/app:latest
          cache-to: type=inline
  ```

BuildKit doesn't preserve `RUN --mount=type=cache` mounts in GitHub Actions
cache by default. To persist them across builds, use the
`reproducible-containers/buildkit-cache-dance` action together with
`actions/cache`.

## Build secrets

A build secret is sensitive data (password, API token) consumed during the
build. First, mount it in the Dockerfile — never bake secrets into layers or
`ARG`s:

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine
RUN --mount=type=secret,id=github_token,env=GITHUB_TOKEN ...
```

Then supply the value via one of three `build-push-action` inputs:

| Input | Source | Use when |
| --- | --- | --- |
| `secrets` | Inline value from the workflow | passing a secret value directly |
| `secret-envs` | Environment variable on the runner | a prior step set an env var |
| `secret-files` | A file on the runner | mounting `.npmrc`, credentials, etc. |

```yaml
      - name: Build
        uses: docker/build-push-action@v6
        with:
          tags: user/app:latest
          secrets: |
            "github_token=${{ secrets.GITHUB_TOKEN }}"
```

Secrets mount as files at `/run/secrets/<id>` by default; add `env=` in the
Dockerfile mount to expose one as an environment variable, or `target=` to
change the path. For multi-line values, wrap the key-value pair in quotes
(escape inner quotes by doubling them). For non-root `RUN` users, set
`uid`/`gid`/`mode` on the mount so the file is readable.

For SSH (e.g. cloning a private repo), use `RUN --mount=type=ssh` in the
Dockerfile and pass `ssh: default` in the action; bootstrap an SSH agent on the
runner first (e.g. via `MrSquaare/ssh-setup-action`).

## Test an image before pushing

Build and `load` the image into the runner, run tests against it, then build and
push. Because the first build is cached, the push step rebuilds only the extra
platforms:

```yaml
env:
  TEST_TAG: user/app:test
  LATEST_TAG: user/app:latest

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and export to Docker
        uses: docker/build-push-action@v6
        with:
          load: true
          tags: ${{ env.TEST_TAG }}

      - name: Test
        run: docker run --rm ${{ env.TEST_TAG }}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ env.LATEST_TAG }}
```

## Rules of thumb

- Always `setup-buildx-action` before `build-push-action`; add
  `setup-qemu-action` for emulated multi-platform builds.
- Gate pushes and registry logins on non-PR events so PRs build but don't publish.
- Generate tags/labels with `metadata-action` rather than hardcoding them.
- Use `cache-to: mode=max` with the `gha` or `registry` backend; `inline` only
  supports `min`.
- Pass secrets through `secrets`/`secret-envs`/`secret-files` and
  `RUN --mount=type=secret` — never via `ARG` or copied files.
- `load: true` works for single-platform locally; multi-platform local loads
  need the containerd image store.
