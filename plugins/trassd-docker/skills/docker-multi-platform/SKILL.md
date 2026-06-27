---
name: docker-multi-platform
description: Building container images for multiple OS/architecture combinations (e.g. linux/amd64, linux/arm64) with Docker Buildx — the --platform flag, buildx builders and drivers, emulation vs native nodes, and cross-compilation. Use when producing multi-arch images, configuring a buildx builder, or choosing a build driver.
---

# Docker multi-platform builds

A multi-platform build is a single build invocation that targets multiple
OS/CPU-architecture combinations, producing one image that runs on
`linux/amd64`, `linux/arm64`, `windows/amd64`, and so on. Containers share the
host kernel, so a `linux/amd64` container can't run on an arm64 host without
emulation — multi-platform images solve this by packaging multiple variants
behind a single manifest list.

When you push a multi-platform image, the registry stores a manifest list
pointing to one manifest per platform. On pull, Docker automatically selects
the variant matching the host's architecture.

## Prerequisites

Multi-platform images need an image store that supports manifest lists.

- Docker Desktop and Docker Engine 29.0+ use the containerd image store by
  default, which supports multi-platform images out of the box — no extra setup.
- On older Docker Engine (or after upgrading from classic storage drivers),
  either enable the containerd image store via the daemon configuration file,
  or create a custom builder using the `docker-container` driver.

## Build command

Use the `--platform` flag to set the target platforms:

```bash
docker buildx build --platform linux/amd64,linux/arm64 .
```

`docker build` is an alias for `docker buildx build`, but `docker build` always
defaults to the bundled `docker` driver builder. To use a custom builder with
`docker build`, pass `--builder` or set `BUILDX_BUILDER`. Prefer
`docker buildx build` when working with custom builders so your selected
builder is interpreted correctly.

## Builders and drivers

A builder is a BuildKit daemon that runs your builds. Docker Engine creates a
default builder bound to the current Docker context, using the `docker` driver.
Buildx supports four drivers:

| Driver | What it is | Multi-arch | Auto-load image | Cache export |
| --- | --- | :---: | :---: | :---: |
| `docker` | BuildKit bundled in the daemon (default) | | ✅ | partial |
| `docker-container` | Dedicated BuildKit container via Docker | ✅ | | ✅ |
| `kubernetes` | BuildKit pods in a Kubernetes cluster | ✅ | | ✅ |
| `remote` | Connects to a manually managed BuildKit daemon | ✅ | | ✅ |

The default `docker` driver can't produce multi-arch images. Use one of the
other drivers — most commonly `docker-container`.

Create and select a `docker-container` builder:

```bash
docker buildx create \
  --name container-builder \
  --driver docker-container \
  --bootstrap --use
```

List builders and see which is selected (the `*` marks the selected builder);
switch with `docker buildx use <name>`:

```bash
docker buildx ls
```

### Loading non-default-driver builds locally

Images built with non-`docker` drivers aren't loaded into the local image store
automatically — with no output specified, the result goes to build cache only.
To load into the local image store, pass `--load`. Note `--load` doesn't work
for multi-platform builds; push such images to a registry directly with
`--push`. You can make a custom builder load by default with the
`default-load` driver option:

```bash
docker buildx create --driver-opt default-load=true
```

## Three strategies for multi-platform builds

Pick based on your use case: emulation (easiest), multiple native nodes
(fastest, more setup), or cross-compilation (fast, language-dependent).

### 1. Emulation with QEMU

Easiest to start with and requires no Dockerfile changes — BuildKit detects the
architectures available for emulation. Docker Desktop supports it by default
using the QEMU bundled in its VM. Official BuildKit releases also bundle QEMU
user-mode emulators, so manual install is usually unnecessary.

Emulation can be much slower than native builds, especially for compile- or
compression-heavy work. Prefer native nodes or cross-compilation when possible.

If a third-party BuildKit package doesn't ship the emulators, install QEMU and
register the executable types on the host (requires Linux kernel 4.8+,
`binfmt-support` 2.1.7+, statically compiled QEMU binaries):

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

Verify registration by checking for the `F` flag in
`/proc/sys/fs/binfmt_misc/qemu-*`.

### 2. Multiple native nodes

Better than emulation for complex cases QEMU can't handle, with better
performance. Add nodes to a builder with `--append`, typically from per-arch
Docker contexts:

```bash
docker buildx create --use --name mybuild node-amd64
docker buildx create --append --name mybuild node-arm64
docker buildx build --platform linux/amd64,linux/arm64 .
```

This adds the overhead of setting up and managing a builder cluster. Docker
Build Cloud is a managed alternative that provides native ARM and x86 builders
plus a shared cache:

```bash
docker buildx create --driver cloud <ORG>/<BUILDER_NAME>
docker build \
  --builder cloud-<ORG>-<BUILDER_NAME> \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  --tag <IMAGE_NAME> \
  --push .
```

### 3. Cross-compilation with multi-stage builds

If your language supports cross-compilation, build for target platforms from
the builder's native architecture — avoiding emulation entirely. BuildKit
provides pre-defined build args including `BUILDPLATFORM`, `TARGETPLATFORM`,
`TARGETOS`, and `TARGETARCH`.

Pin the build stage to the builder's platform with `--platform=$BUILDPLATFORM`
so emulation doesn't kick in, declare the `TARGET*` args you need, and pass
them to the compiler. Go example:

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:alpine AS build
ARG TARGETOS
ARG TARGETARCH
WORKDIR /app
COPY . .
RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -o server .

FROM alpine
COPY --from=build /app/server /server
ENTRYPOINT ["/server"]
```

```bash
docker build --platform linux/amd64,linux/arm64 -t go-server .
```

`ARG` instructions are scoped to the stage that declares them — repeat
`ARG TARGETOS` / `ARG TARGETARCH` in each stage that needs them. The exact
cross-compilation steps vary by language; consult your toolchain's docs. For Go
and other languages, the [`tonistiigi/xx`](https://github.com/tonistiigi/xx)
helper image simplifies cross-compiling.

## Rules of thumb

- Default `docker` driver only builds the host platform; switch to
  `docker-container` (or `kubernetes`/`remote`) for multi-arch.
- Multi-platform builds can't be `--load`ed into the local store — `--push` to
  a registry, or enable the containerd image store to load locally.
- Prefer native nodes or cross-compilation over QEMU for compile-heavy builds.
- For cross-compilation, always pin the build stage with `--platform=$BUILDPLATFORM`.
