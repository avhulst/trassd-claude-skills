---
name: coolify-build-packs
description: Helps pick and configure the right build pack for a Coolify application — Nixpacks, Railpack, Dockerfile, Docker Compose, or Static — and explains when each fits and the key settings (base directory, ports, static mode, build args). Use when deploying an application or choosing how its Docker image is built in Coolify.
---

# Coolify Build Packs

Coolify deploys every application as a Docker container, so it needs a Docker
image. A **build pack** transforms your source code into that image and manages
the build/deploy process. You can also skip building entirely and deploy a
**pre-built Docker image** from a registry.

Use this skill when creating an application resource and choosing how its image is
built, or when a deployment fails because of build-pack misconfiguration (wrong
port, wrong base directory, missing static mode).

## Choosing a build pack

Build packs split into two groups: ones that **auto-generate** a Dockerfile
(Nixpacks, Railpack, Static) and ones that use **your own** build definition
(Dockerfile, Docker Compose).

| Build pack | Best for | Config you write |
| --- | --- | --- |
| **Nixpacks** (default) | Most apps; quick automated builds with minimal config | none (optional `nixpacks.toml`/`.json`) |
| **Railpack** (beta) | Same use cases as Nixpacks; its successor | none (optional `railpack.json`) |
| **Static** | Pre-built static sites / SPAs, no server-side processing | base directory only |
| **Dockerfile** | Full control over the image build | your `Dockerfile` |
| **Docker Compose** | Multi-service / multi-container apps | your `docker-compose.yml` |
| **Docker Image** | Deploy an existing image from a registry | none (image name) |

Rules of thumb:

- Start with **Nixpacks** unless you have a reason not to — Coolify defaults to it
  and auto-detects language and commands (e.g. picks `npm ci` / `yarn install` /
  `pnpm install` from your lock file).
- Reach for **Dockerfile** when you already have one or need precise control.
- Use **Docker Compose** for anything multi-service (app + worker + db) where the
  compose file is the single source of truth.
- Use **Static** when the site is already built and committed to the repo and just
  needs a web server.
- Consider **Railpack** as the actively-developed successor to Nixpacks; it is
  marked Beta in Coolify (functional but may change). Nixpacks is in maintenance
  mode but still the stable default.

Note: Nixpacks, Railpack, Static, Dockerfile, and Docker Compose all build from a
**git repository** (public, GitHub App, or Deploy Key). To deploy a registry
image instead, use the Docker Image deployment type.

## Common configuration

These settings appear across build packs in the application config:

- **Base Directory** — the root Coolify/the builder uses. `/` if your files are at
  the repo root, or a subfolder like `/backend` for monorepos.
- **Port Exposes** — the port(s) the container exposes. The **first port is the
  default for health checks**. Set this to whatever your app listens on (e.g.
  `3000` for Node, `9000` for PHP-FPM, `80` for Nginx). Dockerfile builds default
  to port `3000`.
- **Port Mappings** — map a container port to the host, e.g. `8080:80`. Avoid
  unless needed: mapping to the host loses features like Rolling Updates. Useful
  for very-high-traffic websocket servers that don't need a domain and want to
  skip the proxy.
- **Commands** — override the auto-detected Install / Build / Start commands. Leave
  empty to let the build pack detect them.
- **Force HTTPS** (on by default) and **Auto Deploy** (deploy on push; GitHub App
  repos only).

If the domain shows "No Available Server", the container is usually unhealthy or
the port is wrong — run `docker ps` on the server, and make sure the app listens
on all interfaces (`0.0.0.0`), not just localhost, and that the configured port
matches.

## Nixpacks

Nixpacks (by Railway) inspects your repo, generates a Dockerfile, and builds an
image from it. It deploys both static and non-static apps. Git-based deployments
only.

- **Non-static:** set Base Directory and the **Port** your app listens on; leave
  "Is it a static Site?" unchecked.
- **Static site:** enable **Is it a static Site?** — the port is then forced to
  `80` (intentional, not changeable), and a **Publish Directory** field appears
  (the output dir of your build, commonly `/dist`). Nixpacks builds the site and
  bakes a web server (Nginx) into the image to serve it. The static-site toggle is
  Nixpacks-only.
- **Config file:** a `nixpacks.toml` or `nixpacks.json` at the repo root is picked
  up automatically. You may need this file for custom command overrides to take
  effect.
- **Known issue:** Nixpacks may pull older package versions than you need (common
  with Node.js); pin versions via the config file / `nixpkgs` archive.

## Railpack

Railpack is Railway's successor to Nixpacks. It auto-detects the language,
installs deps, and configures build/start commands with zero config, building an
optimized image with BuildKit. Setup steps mirror Nixpacks — just select
**Railpack** in the build-pack selector. Marked **Beta** in Coolify.

- **Config file:** optional `railpack.json` at the repo root (add
  `"$schema": "https://schema.railpack.com"` for editor autocomplete).
- **Env vars:** prefixed `RAILPACK_*`, e.g. `RAILPACK_BUILD_CMD`,
  `RAILPACK_INSTALL_CMD`, `RAILPACK_START_CMD`, `RAILPACK_PACKAGES`,
  `RAILPACK_BUILD_APT_PACKAGES` / `RAILPACK_DEPLOY_APT_PACKAGES`,
  `RAILPACK_DISABLE_CACHES`, `RAILPACK_VERBOSE`.

Railpack vs Nixpacks at a glance: `railpack.json` (JSON) vs `nixpacks.toml` (TOML);
BuildKit vs native Docker build; Mise vs Nix packages; `RAILPACK_*` vs
`NIXPACKS_*` env prefix; actively developed (Beta) vs maintenance mode (Stable).

## Dockerfile

Use your own `Dockerfile` for complete control over the build. Set Base Directory,
then on the network step set the **port** your app listens on (defaults to `3000`).

- **Pre/Post deployment commands:** optional commands run with `sh -c` —
  pre-deployment runs in the existing container before deploy; post-deployment runs
  in the newly built container after deploy.
- **Build arguments (Advanced menu):**
  - *Inject Build Args to Dockerfile* (enabled by default) — Coolify injects your
    env vars as Docker `ARG`s. Disable to manage `ARG`s yourself.
  - *Include Source Commit in Build* (disabled by default) — exposes
    `SOURCE_COMMIT` as a build arg. Leave disabled to preserve the build cache;
    enabling it invalidates the cache on every commit.

## Docker Compose

Use the Docker Compose build pack for multi-service apps. The
`docker-compose.y[a]ml` file is the **single source of truth** — settings you'd
normally set in the Coolify UI (env vars, storage) must be defined in the compose
file. Set Base Directory and the **Docker Compose Location** (path relative to the
base directory; the file extension must match exactly).

Key behaviors and pitfalls:

- **Networking:** Coolify auto-creates one isolated bridge network per stack
  (named after the resource UUID) and attaches its Traefik proxy to it. Services
  reach each other by service name, e.g. `http://backend:8080`.
- **Do NOT define custom `networks:`.** Defining them puts containers on two
  networks; Traefik may route to the wrong IP, causing intermittent hangs / 504s.
  Remove all `networks:` sections and rely on the auto-created network.
- **Exposing services:** assign a **domain** to a service (add the container port
  if it isn't 80, e.g. `http://example.com:3000`), or add `ports:` to publish on
  the host (which bypasses the proxy — beware exposing private services). With no
  domain and no port, a service stays internal.
- **Cross-stack networking:** enable *Connect to Predefined Network* to reach
  services in another stack, then reference them by full name like
  `postgres-<uuid>`.
- **Magic env vars:** Coolify generates values via `SERVICE_<TYPE>_<IDENTIFIER>`
  (URL, FQDN, USER, PASSWORD, BASE64, HEX, …). Use hyphens not underscores in
  identifiers when a port is involved.
- **Storage extras:** `is_directory: true` tells Coolify to create a bind-mount
  directory; `content:` (or top-level `configs:`) creates a file with content.
- **Healthchecks:** set `exclude_from_hc: true` on run-once services (e.g.
  migrations) so they don't fail the overall healthcheck.
- **Build args:** same *Inject Build Args* / *Include Source Commit* options as the
  Dockerfile build pack.
- **Raw Compose Deployment** deploys the file directly without most Coolify magic —
  advanced users only.

For env-var syntax, magic variables, networking, and storage details, see the
Docker Compose reference: [references/docker-compose.md](references/docker-compose.md).

Minimal Coolify-friendly compose (no custom networks):

```yaml
services:
  frontend:
    image: your-frontend:latest
  backend:
    image: your-backend:latest
    environment:
      - DATABASE_URL=${DATABASE_URL:?}   # required; blocks deploy if unset
```

## Static

Static Build Packs take **already-built** files from your repo and bake them into
an image with a web server (Nginx). They only work if the site is pre-built (e.g.
by Astro, Webstudio, or any static generator) and the output is committed to git.

Steps: select **Static** as the build pack, set the **Base Directory** to where the
built files live (`/` at repo root, or e.g. `/out`), pick the web server (Nginx is
the only option), enter your domain(s) (comma-separated for multiple), and deploy.
You can optionally **Generate** and edit the default web-server config, then
**Restart** for it to take effect.

If your site is built by Coolify rather than pre-built, prefer Nixpacks/Railpack
static mode instead, which builds and then serves the output.
