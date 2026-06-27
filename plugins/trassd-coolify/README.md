# trassd-coolify

Skills and agents that enforce **Coolify** ([coolify.io](https://coolify.io))
best practices: the projects/environments/resources model, application build
packs, build-time vs runtime environment variables & secrets, persistent
storage, domains & TLS, the Caddy/Traefik reverse proxy, managed databases & S3
backups, Git CI/CD with health checks & rolling updates, Docker destinations &
networks, server hardening, and the REST API / MCP integration.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `coolify-concepts` | Servers, projects, environments, resources, containers, reverse proxy, teams |
| `coolify-build-packs` | Nixpacks / Dockerfile / Docker Compose / Static / Railpack — choosing & configuring |
| `coolify-environment-variables` | Build vs runtime vars, Docker build secrets, multiline/literal, shared & magic variables |
| `coolify-persistent-storage` | Volume & file/directory mounts, avoiding data loss on redeploy |
| `coolify-domains-tls` | FQDNs, DNS records, automatic Let's Encrypt TLS |
| `coolify-proxy` | Caddy/Traefik, dynamic config, basic auth, DNS-challenge & wildcard certs |
| `coolify-databases-backups` | Managed databases, database TLS, scheduled S3 backups |
| `coolify-cicd-deployments` | Git auto-deploy, webhooks, preview deploys, health checks, rolling updates |
| `coolify-destinations` | Docker (Standalone/Swarm) destinations and custom Docker networks |
| `coolify-server-hardening` | Non-root user, firewall, OpenSSH, OS patching, automated cleanup |
| `coolify-api-mcp` | REST API authentication/tokens and the MCP integration |

## Agents

| Agent | When to use |
|-------|-------------|
| `coolify-deployment-reviewer` | Review a project's Coolify deployment configuration against best practices. |
| `coolify-server-security-auditor` | Audit a Coolify server and platform setup for security and resilience. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-coolify@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
