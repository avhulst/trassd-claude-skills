---
name: coolify-proxy
description: >-
  Work with Coolify's reverse proxy — choose and switch between Traefik
  (default) and Caddy (experimental), add custom container labels / dynamic
  configuration, protect resources with basic auth, and set up DNS-challenge and
  wildcard Let's Encrypt certificates. Use when customizing routing or the
  proxy, adding basic auth, or issuing wildcard / DNS-challenge certificates.
---

# Coolify Reverse Proxy (Traefik / Caddy)

Coolify supports **Traefik** (default) and **Caddy** (experimental). Traefik is
the team's recommended choice — Caddy needs more manual setup and has fewer
guides. Only pick Caddy if you are comfortable configuring it yourself or need a
specific Caddy feature.

> Caution: incorrect proxy settings can make every app on the server
> inaccessible. Test changes in a non-production environment first.

## Switching proxies

Since `beta.237` you can switch between Caddy and Traefik at any time. For
resources created **before** `beta.237`, ensure they have `caddy_*` or
`traefik_*` labels first:

- Automatic — restarting the resource adds the missing labels.
- Manual — Applications: click **Reset to Coolify Default Labels**; Services:
  just save the service (labels are added automatically).

Then restart the resource so the new labels apply.

## Traefik: customizing a resource

Coolify auto-generates Traefik labels (router, service, entrypoints, TLS
certresolver). You extend behaviour by editing a resource's **Container Labels**.
Available features include basic auth, custom SSL certs, the dashboard, custom
middlewares (rate limiting, IP allowlists, redirects, headers), dynamic
configuration, load balancing, and wildcard certificates.

### Basic auth (Traefik)

Add a `basicauth` middleware label and attach it to the router. Replace
`<random_unique_name>` with a name you invent and `<unique_router_name>` with
the router name Coolify already generated.

```yaml
traefik.http.middlewares.<random_unique_name>.basicauth.users=test:$2y$12$ci.4U63YX83CwkyUrjqxAucnmi2xXOIlEF6T/KdP9824f1Rf1iyNG
traefik.http.routers.<unique_router_name>.middlewares=<random_unique_name>
```

If the router already has a `middlewares` label, **append** rather than replace,
comma-separated: `...middlewares=gzip,<random_unique_name>`.

Generate the credential hash with htpasswd (part of `apache2-utils` on
Debian/Ubuntu):

```bash
htpasswd -nbB test test   # username test, password test
```

For Docker Compose / one-click services, put the same `basicauth.users` label
under the service's `labels:`. Escape special characters (`$`, `@`, `,`) — quote
the value and backslash-escape inside double quotes.

### Dynamic configuration (no restart)

Dynamic configurations apply to Traefik on the fly without restarting it. Add
them under `Server / Proxy → Dynamic Configurations` in the sidebar. Some
dynamic configs are required by Coolify itself and cannot be deleted.

## DNS challenge (Traefik)

By default Traefik uses the **HTTP challenge** (`httpChallenge`), which needs
port 80 publicly reachable. Switch to the **DNS challenge** when you need
wildcard certificates or your server has no public port 80 (firewalled,
internal, Tailscale-only). Traefik (via Lego) creates a temporary
`TXT` record under `_acme-challenge.<domain>` for Let's Encrypt to verify.

Edit the proxy config under **Servers → your server → Proxy**: add a provider
env var and swap the `httpchallenge` command flags for `dnschallenge`. Example
(Cloudflare):

```yaml
services:
  traefik:
    environment:
      - CF_DNS_API_TOKEN=<Cloudflare API Token>
    command:
      # remove the two httpchallenge lines, add:
      - '--certificatesresolvers.letsencrypt.acme.dnschallenge.provider=cloudflare'
      - '--certificatesresolvers.letsencrypt.acme.dnschallenge.delaybeforecheck=0'
```

- Set `provider=<name>` to your provider's Lego identifier and supply that
  provider's required env vars (find both in the Lego DNS provider list). You
  can use `env_file` to keep secrets out of the UI.
- Restart the proxy afterwards.
- If TXT records propagate slowly, raise `delaybeforecheck` (e.g. `30`).
- CNAME-delegated domains failing on renewal: set
  `LEGO_DISABLE_CNAME_SUPPORT=true`.
- While testing, point at the Let's Encrypt staging CA to avoid rate limits, via
  `--certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory`,
  then remove it.

## Wildcard certificates (Traefik)

A wildcard cert (`*.example.com`) covers every subdomain with one cert, so new
deployments are reachable over HTTPS immediately without a per-resource ACME
round trip.

Prerequisites: Traefik must already use the **DNS challenge** (above), and a
wildcard `A` record `*.example.com` must point at the server.

Add these labels to the Traefik proxy config (replace `coolify.io` with your
domain):

```yaml
labels:
  - traefik.http.routers.traefik.tls.certresolver=letsencrypt
  - traefik.http.routers.traefik.tls.domains[0].main=coolify.io
  - traefik.http.routers.traefik.tls.domains[0].sans=*.coolify.io
```

Restart the proxy. Then either set each resource's Domain to a subdomain
(`https://app.coolify.io`) — Traefik reuses the wildcard cert — or, for a
multi-tenant SaaS where one app serves every subdomain, leave the Domain empty
and add custom router labels using a `HostRegexp` rule, e.g. (Traefik v3):

```yaml
traefik.http.routers.<router_https>.rule=HostRegexp(`^.+\.coolify\.io$`)
traefik.http.routers.<router_https>.tls.certresolver=letsencrypt
traefik.http.routers.<router_https>.tls=true
```

Restart the resource for domain changes to take effect.

## Caddy

Caddy auto-manages SSL/TLS via a declarative Caddyfile and includes reverse
proxy, load balancing, and HTTP/2 out of the box. Use Traefik instead if you
need advanced/dynamic routing, middleware, or complex load balancing.

### Basic auth (Caddy)

Hash the password with Caddy's own CLI (other hash methods cause auth failures),
then add it to the application's Caddyfile:

```bash
caddy hash-password --plaintext "your_plaintext_password"
```

```bash
caddy_0.basicauth.<username>="<hashed_password>"
```

Save and restart the application to apply.

### DNS challenge (Caddy)

Same triggers as Traefik (wildcards, no public port 80). Unlike Traefik, Caddy's
DNS provider support **must be compiled into the binary** — the default
`lucaslorentz/caddy-docker-proxy` image has no DNS modules. Use a
`dockerfile_inline` build in the proxy config so the correct binary is built
automatically (no separate build/registry). Example (Cloudflare):

```yaml
services:
  caddy:
    build:
      dockerfile_inline: |
        FROM caddy:2.11-builder AS builder
        RUN xcaddy build --with github.com/lucaslorentz/caddy-docker-proxy/v2@v2.8.11 --with github.com/caddy-dns/cloudflare@v0.2.4
        FROM caddy:2.11-alpine
        COPY --from=builder /usr/bin/caddy /usr/bin/caddy
        CMD ["caddy", "docker-proxy"]
    environment:
      - CF_API_TOKEN=<Cloudflare API Token>
    labels:
      - caddy.acme_dns=cloudflare {env.CF_API_TOKEN}
```

- For other providers, find the module at `github.com/caddy-dns`, update the
  `--with github.com/caddy-dns/<provider>` line, the env var, and the
  `caddy.acme_dns` label.
- Restart the proxy; Caddy builds the image on first start, then uses DNS
  challenge for issuance/renewal.
- Slow propagation: add a per-site `tls { dns <provider> ...; propagation_delay
  30s }` block in `/data/coolify/proxy/caddy/dynamic/Caddyfile`.
- Rate-limit testing: add label
  `caddy.acme_ca=https://acme-staging-v02.api.letsencrypt.org/directory`, remove
  it when done.
- `unknown provider` error → the DNS module wasn't compiled in; fix the `--with`
  line and restart to rebuild.
