---
name: coolify-concepts
description: Explains how a Coolify install is structured — servers, projects, environments, resources, containers, the reverse proxy, and teams — and how a deployment maps across them. Use when getting oriented in Coolify, deciding how to lay out projects and environments, or choosing between Coolify Cloud and self-hosting.
---

# Coolify Concepts

Coolify is an open-source, self-hosted PaaS alternative to platforms like Vercel,
Railway, and Heroku. You bring your own server (a VPS, EC2, Raspberry Pi, or even
an old laptop, reachable over SSH) and Coolify manages deployments on it. It is
not a cloud that hosts your apps for you, and it does not secure or update your
server — that remains your responsibility.

Use this skill to build a correct mental model before laying out projects and
environments, or when deciding how to run Coolify itself.

## The object hierarchy

Coolify organizes everything into a fixed hierarchy. Lay your setup out along
these boundaries.

- **Server** — the machine that provides compute. Physical or rented; Coolify
  connects to it over SSH. A single Coolify instance can manage multiple servers
  (single-server, multi-server, and Docker Swarm setups are supported).
- **Project** — the top-level structure. A project groups environments and
  resources. Run many projects on the same server. Rule of thumb: one project per
  product or context (e.g. one for hobby resources, one for work).
- **Environment** — a tailored setup *within* a project that determines how its
  resources run, e.g. `development` for testing and `production` for the live
  app. Multiple environments can live on a single server, letting you switch
  between them.
- **Resource** — an application or service you deploy: a website, API, database,
  etc. Each resource has its own configuration (domains, backups, health checks).
  Resources are either your own apps or pre-set **one-click services**.

```
Server
└── Project
    └── Environment (development / production / …)
        └── Resource (app, database, service)
            └── Container (running instance)
```

A common layout: one project per product, a `production` and a `development`
environment inside it, and the app plus its database as resources in each
environment.

## Containers — everything runs as Docker

Every resource Coolify deploys runs as a **Docker container**, isolating each app.
To run a container you need a Docker image, obtained one of two ways:

- **Build on Coolify** from your source code (Coolify auto-builds from a
  Dockerfile or Docker Compose file). Building is resource-intensive and needs a
  capable server.
- **Use a pre-built image** — pull from a public registry like Docker Hub or
  GitHub Container Registry, or build elsewhere, push to a registry, and let
  Coolify deploy it as a container.

When choosing *how* an app's image gets built, see the `coolify-build-packs`
skill.

## Reverse proxy and SSL

A reverse proxy sits between users and your containers, forwarding each request to
the right container. Coolify ships two proxy options — **Caddy** and **Traefik**.
This is what lets you run many apps on one server without juggling ports: each app
gets its own domain (Coolify supports unlimited domains), e.g. 20 apps each on a
unique domain.

The proxy also manages SSL/TLS automatically. Enter a domain with `https://` and
the proxy requests and installs a Let's Encrypt certificate with no manual
configuration, renewing it automatically before expiry.

## Teams

Coolify supports multiple users and teams; each team can own its own projects and
environments, and you can assign roles such as admin. Note the docs flag the teams
feature as not yet fully polished for production use.

## Security is your responsibility

Coolify simplifies deployment management but does **not** manage your server's
security or OS updates. Keeping the underlying server secure and patched is up to
you regardless of how you run Coolify.

## Coolify Cloud vs. self-hosted

Both options give the full feature set — Coolify does not lock features behind a
paywall. The difference is who runs the Coolify instance.

| | Coolify Cloud | Self-Hosted |
| --- | --- | --- |
| Coolify instance | Hosted & updated by the Coolify team | You install, run, and update it |
| Server resources for Coolify | None needed (runs on their infra) | You allocate some of your server |
| Cost | Paid, from ~$5/month | Free (you pay only for your server) |
| Updates | Tested by the core team, may lag slightly | You test and upgrade yourself |
| Support / backups / email | Managed, pre-configured | You set up yourself |

In both cases you still bring and are responsible for your own servers and any
other services on them. Choose **Cloud** for a quick managed setup; choose
**self-hosted** for full control and lowest cost.
