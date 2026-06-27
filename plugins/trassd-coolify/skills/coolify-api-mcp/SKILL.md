---
name: coolify-api-mcp
description: >-
  Automate Coolify programmatically — authenticate to the REST API with Bearer
  tokens, pick the right token permissions, handle team scoping, rate limits and
  IP allowlisting, and wire up Coolify's built-in MCP server for AI tools like
  Claude Code or Cursor. Use when scripting against the Coolify API, debugging
  401/403/429 responses, choosing token permissions, or connecting an AI
  assistant to a Coolify instance over MCP.
---

# Coolify API & MCP

## REST API authentication

Coolify authenticates API requests with **Bearer tokens**. Each token is scoped
to a single team and carries specific permissions. Coolify validates the token,
identifies the user/team, then checks permissions for the requested action.

### Enable the API

**Settings → Advanced → API Settings → toggle API Access on.** Optionally set
**Allowed IPs** (leave empty or `0.0.0.0` to allow all — not recommended in
production). With a `root` token you can also toggle it via the API:

```bash
curl -X POST https://<your-coolify>/api/v1/enable  -H "Authorization: Bearer <token>"
curl -X POST https://<your-coolify>/api/v1/disable -H "Authorization: Bearer <token>"
```

### Create a token

**Security → API Tokens** → name it, choose expiration (7/30/60/90 days, 1 year,
or none), select permissions, **Create**. The token is shown **only once** —
copy it immediately. It looks like `67|abcthisisa123dummytoken`: the part before
`|` is the token **ID**, the rest is the **secret**; both are required.

### Make requests

Base URL is `https://<your-coolify>/api/v1`. Send the token in every request:

```bash
curl https://<your-coolify>/api/v1/teams \
  -H "Authorization: Bearer 67|abcthisisa123dummytoken"
```

Only `/api/health` and `/api/feedback` sit outside the `/v1` prefix.

### Permissions

Grant the least privilege the integration needs (a monitoring dashboard:
`read`; a CI/CD pipeline: `read` + `deploy`; reserve `root` for admin
automation).

| Permission | Access |
|---|---|
| `read` | View servers, projects, applications, databases, services |
| `read:sensitive` | `read` plus passwords, private keys, env vars, logs (otherwise these fields are redacted) |
| `write` | Create, update, delete resources |
| `deploy` | Trigger deployments and manage deploy webhooks |
| `root` | Bypasses all checks; full control, incl. enabling/disabling the API |

`root` can only be granted by **Admin/Owner** roles. Each endpoint requires a
specific permission; a missing one returns **403 Forbidden** listing what's
needed. `root` bypasses all permission checks.

### Team scoping, rate limits, IP allowlisting

- **Team scoping** — a token only sees resources of the team that was active
  when it was created. Need cross-team access → create one token per team. A
  leaked token cannot reach other teams' resources.
- **Rate limit** — 200 requests/minute by default (returns **429** when
  exceeded); configurable via the `API_RATE_LIMIT` env var.
- **IP allowlisting** — **Settings → Advanced → Allowed IPs for API Access**,
  comma-separated IPv4/IPv6 and CIDR ranges. Empty or `0.0.0.0` allows all.
  Include your current IP before saving or you can lock yourself out. Redundant
  entries already covered by a CIDR are deduplicated.

### Security notes

Tokens are stored as **SHA-256 hashes** and cannot be retrieved after creation —
if lost, create a new one. Set expirations for automated systems (Coolify emails
a warning before expiry). Revoke instantly by deleting the token under
**Security → API Tokens**.

### Troubleshooting

- **403 Forbidden** — API not enabled, IP not allowlisted, or token lacks the
  endpoint's required permission.
- **401 Unauthenticated** — wrong/expired token, missing ID+secret, or header
  not exactly `Authorization: Bearer <token>`.
- **Missing resources** — they belong to another team; create a token for that
  team.
- **Sensitive data redacted** — token needs `read:sensitive` or `root`.
- **429 Too Many Requests** — space out requests or raise `API_RATE_LIMIT`.

## MCP server

Coolify ships a built-in **MCP (Model Context Protocol)** server so AI
assistants (Claude Code, Cursor, etc.) can query the instance. It is currently
**read-only** (write operations are planned). All data is scoped to the API
token's team.

### Setup

1. **Enable API access first** (above) — MCP authenticates with API tokens.
2. **Enable MCP** — **Settings → Advanced → MCP Server → Enable MCP Server**, or
   via API with a `root` token:

   ```bash
   curl -X POST https://<your-coolify>/api/v1/mcp/enable  -H "Authorization: Bearer <token>"
   curl -X POST https://<your-coolify>/api/v1/mcp/disable -H "Authorization: Bearer <token>"
   ```

   The endpoint becomes available at `https://<your-coolify>/mcp`.
3. **Create a token** with **only `read`** permission — since MCP is read-only,
   extra permissions add risk without benefit.
4. **Configure the AI client.** Transport is **Streamable HTTP** (not SSE or
   stdio). Provide:
   - URL: `https://<your-coolify>/mcp`
   - Header: `Authorization: Bearer <token>`

   Any MCP client supporting Streamable HTTP can connect.

### Available tools (10, all read-only)

`get_infrastructure_overview` (start here for a bird's-eye summary), then
`list_servers` / `get_server`, `list_projects`, `list_applications` /
`get_application`, `list_databases` / `get_database`, `list_services` /
`get_service`. Detail tools take a resource **UUID**; `list_applications` is
filterable by tag.

All `list_*` tools paginate: `page` (default 1) and `per_page` (default 50, max
100). Responses include a `_pagination.next` value.

### Security and troubleshooting

The AI client can see infrastructure metadata (server IPs, app names, domains,
env-var names). Revoking the token cuts off MCP access immediately.

- **404** — MCP server not enabled (Settings → Advanced).
- **Auth fails** — token revoked/wrong, API access disabled, or token lacks
  `read`.
- **Can't connect** — wrong endpoint URL, instance unreachable from the client,
  or transport not set to Streamable HTTP.
- **Missing resources** — scoped to the token's team; run
  `get_infrastructure_overview` to see what the token can view.
