---
name: docker-compose-env-secrets
description: Configuring environment variables and secrets in Docker Compose the right way using env_file, variable interpolation and precedence, and the secrets mechanism instead of committing credentials. Triggers when setting environment variables or secrets in a compose file.
---

# Compose environment variables and secrets

Guidance for configuring environment and sensitive data in Compose files, based
on the Compose environment-variable and secrets how-tos.

## Two distinct mechanisms

- **Interpolation** inserts variable values *into the Compose file itself* at
  runtime (for example, choosing an image tag).
- **Container environment** (`environment` / `env_file`) sets variables *inside
  the running container*.

These can interact — an interpolated value can be passed through into the
container environment — but they are separate features with separate precedence
rules.

## Variable interpolation in the Compose file

Use `${VAR}` (or `$VAR`) to substitute values into the Compose file. Verify the
resolved result with `docker compose config`.

```yaml
services:
  web:
    image: "webapp:${TAG}"
```

Interpolation applies to unquoted and double-quoted values. Supported braced
forms:

- `${VAR}` — value of `VAR`.
- `${VAR:-default}` / `${VAR-default}` — default if `VAR` is unset/empty
  (`:-`) or just unset (`-`).
- `${VAR:?error}` / `${VAR?error}` — exit with an error if `VAR` is unset/empty
  (`:?`) or just unset (`?`). Use this to fail fast on required variables.
- `${VAR:+replacement}` / `${VAR+replacement}` — use `replacement` only when
  `VAR` is set.

Notes:

- Interpolation can be nested, for example `${VAR:-${FOO:-default}}`.
- Extended shell features like `${VAR/foo/bar}` are **not** supported.
- If a variable can't be resolved and has no default, Compose warns and
  substitutes an empty string — for example `postgres:${TAG}` becomes
  `postgres:` (an invalid image reference). Prefer defaults or the required-value
  form for anything load-bearing.
- Use `$$` for a literal dollar sign; this also stops Compose from interpolating
  a value you want passed through to the container's shell.
- Interpolation applies to YAML values, not keys. For `labels`/`environment`,
  use the equals-sign list form to interpolate a key:
  `- "$VAR_INTERPOLATED_BY_COMPOSE=BAR"`.

### Sources for interpolation values (precedence, highest first)

1. Variables from your shell environment.
2. Variables from a file given with `--env-file`.
3. If `--env-file` is not set, an `.env` file in the project directory.

The project directory is `--project-directory` if set, otherwise the directory
of the first `-f`/`--file` Compose file, otherwise `PWD`. Inspect what Compose
uses with `docker compose config --environment`.

## The `.env` file

The `.env` file is the default way to set interpolation variables. Place it at
the project root next to `compose.yaml`.

```bash
# .env
TAG=v1.5
```

Key syntax rules:

- `#` lines and blank lines are ignored. Key/value separator is `=` or `:`.
- Surrounding spaces are stripped; values may be quoted.
- Unquoted and double-quoted values are interpolated; single-quoted values are
  literal (`VAR='${OTHER}'` stays `${OTHER}`).
- Inline comments need a leading space for unquoted values
  (`VAR=VAL # comment`); for quoted values the comment follows the closing quote.
- Double-quoted values support escapes like `\n`, `\t`, `\\`; single-quoted
  values can span multiple lines.

Override the default file with `--env-file` (relative to the working directory),
which is useful for per-environment files:

```bash
docker compose --env-file ./config/.env.dev up
```

Multiple `--env-file` options are read in order; later files override earlier
ones. Substitution from `.env` files is a Compose CLI feature and is not
supported by `docker stack deploy` (Swarm).

## Setting the container environment

Two attributes set variables inside the container:

- `environment` — inline key/values in the Compose file.
- `env_file` — load values from a file.

A value can be interpolated from the shell or an `.env` file before being set in
the container, for example `environment: [ "DEBUG=${COMPOSE_DEBUG}" ]`.

### Container environment precedence (highest first)

1. `docker compose run --env` on the CLI.
2. `environment` or `env_file` whose value is **interpolated** from the shell or
   an env file.
3. `environment` attribute with a literal value.
4. `env_file` attribute.
5. Image `ENV` directive — only used when none of the above set the variable.

So a literal `environment:` entry beats `env_file`, and both beat the image's
`ENV`. When `environment` and `env_file` set the same variable to literal
values, `environment` wins.

```yaml
services:
  webapp:
    image: webapp
    env_file:
      - ./webapp.env       # NODE_ENV=test
    environment:
      - NODE_ENV=production # wins → NODE_ENV=production
```

Choose per-environment env files (development, testing, production) rather than
editing the Compose file for each. You can also override values from the CLI for
temporary changes.

## Secrets instead of credentials in env vars

Do not commit passwords, certificates, or API keys into the Compose file,
`environment`, or source. Environment variables are exposed to all processes,
are hard to track, and can leak into logs. Use the `secrets` mechanism instead.

Secrets are mounted as files at `/run/secrets/<secret_name>` inside the
container, which permits granular access control via filesystem permissions.
Setup is two steps: define the top-level `secrets`, then reference them per
service with the service-level `secrets` attribute (access is per-service).

```yaml
services:
  myapp:
    image: myapp:latest
    secrets:
      - my_secret
secrets:
  my_secret:
    file: ./my_secret.txt
```

### `_FILE` convention for passwords

Many official images (for example `mysql`, `postgres`) read a credential from a
file path given in a `*_FILE` variable. Point that variable at the mounted
secret rather than putting the password in `environment`:

```yaml
services:
  db:
    image: mysql:latest
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_root_password
      - db_password
secrets:
  db_password:
    file: db_password.txt
  db_root_password:
    file: db_root_password.txt
```

### Build-time secrets

Secrets can also be made available during a build. Source the value from an
environment variable instead of a file so it isn't committed:

```yaml
services:
  myapp:
    build:
      context: .
      secrets:
        - npm_token
secrets:
  npm_token:
    environment: NPM_TOKEN
```
