---
name: docker-multi-stage-builds
description: Use multi-stage builds to ship small production images — separate build and runtime stages, COPY --from, named/target stages, and external base stages. Use when reducing image size or splitting build-time tooling from the runtime image.
---

# Docker multi-stage builds

Guidance for using multi-stage builds to keep build-time tooling out of the
final image. Apply when reducing image size or separating the build of an image
from its runtime output.

## Why multi-stage

A multi-stage build uses multiple `FROM` statements in one Dockerfile. Each
`FROM` starts a new stage and can use a different base. You selectively copy
artifacts from one stage into another, leaving behind everything you don't want
in the final image. This produces a clean separation between building the image
and its final output, so the result contains only the files needed to run the
application. Using multiple stages can also build more efficiently by executing
independent stages in parallel.

## A basic two-stage build

The first stage builds the binary; the second stage copies just that binary
into a minimal base. None of the build tools end up in the final image.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.21 AS build
WORKDIR /src
COPY . .
RUN go build -o /bin/hello ./main.go

FROM scratch
COPY --from=build /bin/hello /bin/hello
CMD ["/bin/hello"]
```

The second `FROM scratch` starts a fresh stage, and `COPY --from=build` pulls
only the built artifact across. The Go SDK and intermediate artifacts are left
behind. You still build with a single `docker build` — no separate build
script.

## Name your stages

By default stages are referenced by integer index, starting at `0` for the
first `FROM` (`COPY --from=0`). Prefer naming stages with `AS <name>` and
referencing the name. Names survive reordering of instructions, so a `COPY`
won't silently break if stages move.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.21 AS build
WORKDIR /src
COPY . .
RUN go build -o /bin/hello ./main.go

FROM scratch
COPY --from=build /bin/hello /bin/hello
CMD ["/bin/hello"]
```

## Pick a build base and a slimmer runtime base

Consider two base images: one for building and unit testing (with compilers,
build systems, and debugging tools), and a separate, typically slimmer base for
production. Late in development the runtime image usually doesn't need build
tooling, and dropping it considerably lowers the attack surface and image size.

## Stop at a specific stage with --target

You don't have to build every stage. Use `--target` to build up to a named
stage — useful for debugging a stage, keeping a `debug` stage alongside a lean
`production` stage, or running a `testing` stage seeded with test data.

```console
$ docker build --target build -t hello .
```

With BuildKit, only the stages the target depends on are built; unrelated
stages are skipped. (The legacy builder processes every stage up to the target
even if the target doesn't depend on them, so prefer BuildKit.)

## Use an external image as a stage

`COPY --from` isn't limited to stages you defined — you can copy directly from
an external image by name, tag, or ID. Docker pulls the image if needed.

```dockerfile
COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf
```

## Use a previous stage as a new stage

A later `FROM` can build on an earlier stage by referencing its name. This
shares common setup across derivative stages without repeating it.

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:latest AS builder
RUN apk --no-cache add build-base

FROM builder AS build1
COPY source1.cpp source.cpp
RUN g++ -o /binary source.cpp

FROM builder AS build2
COPY source2.cpp source.cpp
RUN g++ -o /binary source.cpp
```

## Create reusable base stages

If several images share a lot, factor the shared components into a common base
stage and base your unique stages on it. Docker builds the common stage once, so
derivative images use host memory more efficiently and load faster — and a
single shared base is easier to maintain than several near-duplicate stages
("Don't repeat yourself").
