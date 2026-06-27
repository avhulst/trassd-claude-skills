---
name: coolify-environment-variables
description: >-
  Manage environment variables and secrets in Coolify — build-time vs runtime
  flags, Docker BuildKit build secrets for sensitive values, multiline and
  literal variables, team/project/environment shared variables, and
  predefined/magic variables. Use when configuring env vars or secrets for a
  Coolify application or service.
---

# Coolify Environment Variables

Define environment variables per resource; they become available to your
application. Preview deployments can carry different values, so a preview can
act as a staging environment.

## Two views

- **Normal view** (default): one form card per variable, with key/value fields
  and the `Build Variable`, `Multiline`, and `Literal` checkboxes. Use this for
  per-variable options, multiline values, and locked secrets.
- **Developer view**: a plain-text `.env` editor (`KEY=VALUE`, one per line) for
  bulk editing or pasting an existing `.env`. On save Coolify parses the text
  and creates/updates/removes variables; order is preserved. Lines starting with
  `#` are comments and ignored.

Developer view cannot edit everything:

- Locked secrets show as `KEY=(Locked Secret, delete and add again to change)` —
  delete and re-add to change them.
- Multiline variables show as `KEY=(Multiline environment variable, edit in
  normal view)` — edit them in Normal view.

## Build-time vs runtime

Each variable has two independent flags controlling **when** it is available:
**Build Variable** and **Runtime Variable**. Both default to on.

| Configuration | Build phase | Running container |
|---|---|---|
| Build + Runtime (default) | Available | Available |
| Build only | Available | Not available |
| Runtime only | Not available | Available |

- **Build variables** are injected during the image build. Dockerfile builds add
  them as `ARG`; Docker Compose and Nixpacks/Buildpack builds pass them via
  `--env-file`. They are stored in a separate file (`/artifacts/build-time.env`)
  outside the build context, so they are not baked into the final image.
- **Runtime variables** live in the running container. Coolify writes a `.env`
  loaded by Docker Compose via `env_file` at container start.

Rule of thumb: if a value is only read at startup (e.g. an API key), disable
`Build Variable` to keep it out of the build phase entirely.

## Docker build secrets (for sensitive build-time values)

Plain build variables are passed as `--build-arg`, which records them in image
metadata — anyone with the image can read them via `docker history`. For
sensitive build-time values (private registry tokens, build-time API keys),
enable **Use Docker Build Secrets** in the application's env var settings. This
uses Docker BuildKit (requires Docker 18.09+) to mount secrets into build steps
instead of embedding them in layers.

When enabled, Coolify automatically:

1. Passes build variables via `--secret id=KEY,env=KEY` instead of `--build-arg`.
2. Adds a `# syntax=docker/dockerfile:1` directive to your Dockerfile if missing.
3. Injects `--mount=type=secret` into every `RUN` instruction, exposing each
   secret as an environment variable for that step.
4. Keeps secrets out of image layers and out of `docker history`.

For Docker Compose builds, Coolify adds a native `secrets:` section to the
compose file instead. You do not edit the Dockerfile or compose file yourself.

| | Build args (default) | Build secrets |
|---|---|---|
| Docker flag | `--build-arg KEY=value` | `--secret id=KEY,env=KEY` |
| Visible in `docker history` | Yes | No |
| Stored in image layers | Yes | No |
| Requires BuildKit | No | Yes (Docker 18.09+) |

Build cache: Coolify derives `COOLIFY_BUILD_SECRETS_HASH` from the secret
values, so cache is preserved while secrets are unchanged and invalidated when
they change.

> If BuildKit is unavailable on the build server, Coolify falls back to plain
> `--build-arg` even with this setting on.

## Multiline variables

Enable `Multiline` (Normal view) to preserve line breaks and special characters
— SSH private keys, TLS/SSL certificates, multi-line config or scripts.
Multiline values are wrapped in single quotes at deploy time (no shell
interpretation). In Docker builds they are declared as `ARG KEY` without an
inline value and supplied separately via `--build-arg` so they don't break
Dockerfile syntax. Multiline variables are editable only in Normal view.

## Literal variables

By default Coolify interpolates references like `$OTHER_VAR` inside a value.
Enable `Literal` (Normal view) to treat the whole value as plain text, preserving
`$` and other shell-special characters:

- passwords containing `$` (e.g. `P@ss$word123`)
- regex patterns (e.g. `^user\d+$`)
- templating or literal shell expressions

The `Literal` checkbox is hidden when `Multiline` is on, since multiline values
are always treated literally.

## Shared variables

Three scopes, each set on its own page and referenced with a `{{...}}` token
(keep the literal prefix — do not substitute your real team/project/environment
name):

| Scope | Set on | Reference |
|---|---|---|
| Team | Team page | `{{team.NODE_ENV}}` |
| Project | Projects page → gear icon | `{{project.NODE_ENV}}` |
| Environment (production, staging, …) | Environments page (within a project) → gear icon | `{{environment.NODE_ENV}}` |

Define once (e.g. `NODE_ENV=production`) and reference the token anywhere.

## Predefined variables

Coolify predefines values you can reference from your own variables:

```bash
MY_VARIABLE=$SOURCE_COMMIT   # MY_VARIABLE gets the source commit hash
```

Application variables:

- `COOLIFY_FQDN` — fully qualified domain name(s) of the application
- `COOLIFY_URL` — URL(s) of the application
- `COOLIFY_BRANCH` — source branch name
- `COOLIFY_RESOURCE_UUID` — unique resource identifier
- `COOLIFY_CONTAINER_NAME` — generated container name
- `SOURCE_COMMIT` — source commit hash (by default excluded from Docker builds
  to preserve cache; enable "Include Source Commit in Build" in General settings
  if the build needs it)
- `PORT` — defaults to the first port in `Port Exposes` if unset
- `HOST` — defaults to `0.0.0.0` if unset

Service stack variable:

- `SERVICE_NAME_<ID>` — the service name of a stack service (e.g. `SERVICE_NAME_WEB`
  for a service named `web`). Useful for preview deployments where names vary.

## Magic environment variables (Docker Compose / service stacks)

For compose/service-stack deployments, Coolify auto-generates dynamic values via
`SERVICE_<TYPE>_<IDENTIFIER>`. Generated values are reused across services in a
stack and persist between deployments.

| Type | Generates |
|---|---|
| `SERVICE_URL_<ID>` | URL from the wildcard domain (e.g. `http://app-x.example.com`) |
| `SERVICE_URL_<ID>_3000` | URL routed to a specific port |
| `SERVICE_URL_<ID>=/api` | URL with a path appended |
| `SERVICE_FQDN_<ID>` | FQDN portion of the URL (port/path variants like URL) |
| `SERVICE_USER_<ID>` | Random 16-char string |
| `SERVICE_PASSWORD_<ID>` | Random password, no symbols (`_64_` for 64 chars) |
| `SERVICE_PASSWORDWITHSYMBOLS_<ID>` | Random password with symbols (`_64_` variant) |
| `SERVICE_BASE64_<ID>` | Random string, 32 chars (`_64_`, `_128_` variants) |
| `SERVICE_REALBASE64_<ID>` | Base64-encoded random string, 32 chars (`_64_`, `_128_` variants) |
| `SERVICE_HEX_32_<ID>` | Hexadecimal random string (`_64_`, `_128_` variants) |

Example, in a compose env block:

```yaml
environment:
  - DB_PASSWORD=$SERVICE_PASSWORD_DB
  - APP_URL=$SERVICE_URL_APP
```

> The docs note `SERVICE_BASE64_*` produces a random string that is "not
> Base64-encoded"; use `SERVICE_REALBASE64_*` when you specifically need
> Base64-encoded output.
