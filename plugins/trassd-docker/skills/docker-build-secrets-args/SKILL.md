---
name: docker-build-secrets-args
description: Pass credentials and configuration into Docker builds safely — build secrets via RUN --mount=type=secret, SSH mounts, build ARG vs runtime ENV, and never baking secrets or tokens into image layers. Use when a build needs credentials/tokens (private artifact servers, private Git repos) or parameterized build variables like versions.
---

# Build secrets and build variables

A build secret is any sensitive value (password, API token, SSH key) used
during the build. The core rule: **never pass secrets through `ARG` or `ENV`,
and never bake them into a layer** — both persist in the final image. Use secret
mounts or SSH mounts, which expose the value only for the duration of a single
build instruction.

## Why ARG and ENV leak

- `ENV` values are passed into the build environment **and persist in
  containers** created from the image.
- `ARG` values are not in the image filesystem by default, but they may persist
  in image metadata — provenance attestations and `docker history`.

Both are therefore unsuitable for secrets. Build arguments and environment
variables are for *configuration*, not credentials.

## Build secrets: `RUN --mount=type=secret`

Using a secret is two steps: pass it to the build, then consume it in the
Dockerfile.

Pass it with `docker build --secret`:

```bash
docker build --secret id=aws,src=$HOME/.aws/credentials .
```

Consume it with `--mount=type=secret` on a `RUN`. By default the secret is
mounted as a file at `/run/secrets/<id>`:

```dockerfile
RUN --mount=type=secret,id=aws \
    AWS_SHARED_CREDENTIALS_FILE=/run/secrets/aws \
    aws s3 cp ...
```

### Sources: file or environment variable

A secret's source can be a file (`src=`) or an environment variable (`env=`);
the CLI detects the type automatically. To pass an env var as a secret:

```bash
docker build --secret id=kube,env=KUBECONFIG .
```

When the secret comes from an env var, omitting `env` binds it to a file named
after the variable — `--secret id=API_TOKEN` mounts to `/run/secrets/API_TOKEN`.

### Customizing how it's mounted

- `target=` — mount the secret file at a custom path:

  ```dockerfile
  RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
      aws s3 cp ...
  ```

- `env=` — expose the secret as an environment variable instead of a file:

  ```dockerfile
  RUN --mount=type=secret,id=aws-key-id,env=AWS_ACCESS_KEY_ID \
      --mount=type=secret,id=aws-secret-key,env=AWS_SECRET_ACCESS_KEY \
      aws s3 cp ...
  ```

`target` and `env` can be combined to mount as both a file and a variable.

### Secrets and the build cache

Secret **contents are not part of the build cache** — changing a secret's value
does not invalidate the cache. But secret **properties** (the ID, mount paths)
do participate in the checksum and invalidate when changed. To force a rebuild
after rotating a secret value, pass an extra build arg you also change:

```dockerfile
FROM alpine
ARG CACHEBUST
RUN --mount=type=secret,id=TOKEN,env=TOKEN \
    some-command ...
```

```bash
TOKEN="tkn_pat123456" docker build --secret id=TOKEN --build-arg CACHEBUST=1 .
```

## SSH mounts and private Git

When the credential is an SSH agent socket or key — typically for cloning
private Git repos — use an SSH mount rather than a secret mount:

```bash
docker buildx build --ssh default .
```

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine
ADD git@github.com:me/myprivaterepo.git /src/
```

### Git auth for remote contexts

For HTTP-authenticated private Git (remote context or `ADD` from a private
repo), BuildKit has two predefined secrets:

- `GIT_AUTH_TOKEN` — Basic auth with a fixed `x-access-token` user (GitHub
  style).
- `GIT_AUTH_HEADER` — the raw `Authorization` header value you provide (any
  provider).

```bash
# GitHub
GIT_AUTH_TOKEN=$(gh auth token) docker build \
  --secret id=GIT_AUTH_TOKEN \
  https://github.com/example/todo-app.git

# GitLab CI, custom header
export GIT_AUTH_HEADER="Basic $(echo -n "gitlab-ci-token:${CI_JOB_TOKEN}" | base64)"
docker build --secret id=GIT_AUTH_HEADER https://gitlab.com/example/todo-app.git
```

Set these per host by appending the hostname to the secret ID, e.g.
`id=GIT_AUTH_TOKEN.github.com`. For `COPY`/`ADD` over HTTP, the equivalents are
`HTTP_AUTH_TOKEN_<host>` and `HTTP_AUTH_HEADER_<host>`.

## Build variables: ARG vs ENV (for configuration)

Use these to parameterize a build — never for secrets.

**`ARG`** declares variables for the Dockerfile itself, set at build time. It
has no effect unless referenced by an instruction. Common use: pinning versions
at the top of the file.

```dockerfile
# syntax=docker/dockerfile:1
ARG NODE_VERSION="20"
ARG ALPINE_VERSION="3.20"
FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS base
```

```bash
docker build --build-arg NODE_VERSION=current .
```

**`ENV`** sets variables available to all subsequent instructions in the stage
**and** persists in containers run from the image — useful for runtime config
like `NODE_ENV=production`. You cannot set an `ENV` at build time directly;
combine it with an `ARG` to make it overridable:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20
ARG NODE_ENV=production
ENV NODE_ENV=$NODE_ENV
```

```bash
docker build --build-arg NODE_ENV=development .
```

Because `ENV` persists into runtime, be careful — it can cause unintended
side-effects on the application.

### Scoping in multi-stage builds

A global `ARG` declared before the first `FROM` is **not** automatically
inherited into stages. To use it inside a stage, re-declare it there:

```dockerfile
# syntax=docker/dockerfile:1
ARG NAME="joe"

FROM alpine
ARG NAME              # consume the global arg into this stage
RUN echo $NAME
```

Once an arg is declared or consumed in a stage, it's inherited by child stages.

### Predefined args

- **Platform args** (`TARGETPLATFORM`, `TARGETOS`, `TARGETARCH`,
  `BUILDPLATFORM`, …) are available in the global scope for cross-compilation;
  declare them with `ARG` inside a stage to use them.
- **Proxy args** (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, `FTP_PROXY`,
  `ALL_PROXY`) need no declaration — passing `--build-arg HTTP_PROXY=...` is
  enough. They're excluded from the cache and `docker history` by default,
  *unless* you reference them in the Dockerfile (then proxy config lands in the
  cache).
