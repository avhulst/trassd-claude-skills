---
name: coolify-cicd-deployments
description: Wire up Git-based continuous deployment in Coolify — connect GitHub/GitLab/Bitbucket/Gitea or any Git provider, auto-deploy on push (via GitHub App, GitHub Actions, or webhooks), set up PR preview deployments, and configure health checks and zero-downtime rolling updates. Use when setting up auto-deploy, webhooks, preview environments, or deployment health/rollout behavior.
---

# Coolify CI/CD & Deployments

Coolify deploys applications directly from Git repositories. Once connected, it
pulls source, builds a Docker image via the chosen build pack, deploys the
container, and (if auto-deploy is on) watches the repo and redeploys on new
commits. Use this skill when wiring up automatic deploys, webhooks, preview
environments, or deployment health/rollout behavior.

## Connecting a Git provider

Coolify works with **any** Git provider — GitHub, GitLab, Bitbucket, Gitea,
Gogs, Forgejo, self-hosted servers, and more. Access methods:

- **Public repository** — provide the HTTPS URL; no authentication needed. Works
  with any provider.
- **Git Provider App integration** — available for **GitHub**. Recommended for
  supported providers: full integration with automatic webhooks, pull-request
  deployments, commit status updates, and no SSH key management.
- **Deploy keys (SSH)** — work with any provider. More manual webhook setup, but
  ideal for air-gapped, restricted, or custom/self-hosted Git servers.

To deploy your own app **without** Git (e.g. a prebuilt image or a Docker Compose
file you upload), use a Service instead.

## Automatic deployments

Three ways to trigger an auto-deploy on push (GitHub):

1. **GitHub App** — after setting up the GitHub App, Coolify usually enables
   **Auto Deploy** automatically. If not, enable it under the application's
   **Advanced** page → general section.
2. **GitHub Actions** — build on GitHub, push the image to a registry (GHCR,
   Docker Hub), and have Coolify deploy it.
3. **Webhooks** — wire a repository webhook to Coolify (below).

### Setting up a push webhook

1. **Enable Auto Deploy**: application config → **Advanced** page → enable
   **Auto Deploy** in the general section.
2. **Set a webhook secret** in Coolify (a random string) and copy the **webhook
   URL** Coolify shows.
3. On the repo: **Settings → Webhooks → Add webhook**, then:
   - **Payload URL** = the Coolify webhook URL.
   - **Secret** = the same secret you set in Coolify.
   - Enable **SSL verification**.
   - Select **Just the `push` event**.
   - Enable **Active** and save.

> [!IMPORTANT]
> The webhook secret acts like a password — Coolify only accepts the webhook if
> the secret matches. Keep the webhook URL and secret safe.

## Preview deployments

Preview deployments spin up an isolated environment for each pull request, each
with its own URL. They are automatically deleted when the PR is merged or closed.

### Preview URL template and DNS

Each preview gets a URL from a template you set:

- `{{random}}` — a random subdomain per PR deploy.
- `{{pr_id}}` — uses the pull request ID as the subdomain.

> [!IMPORTANT]
> Set up a **wildcard `A` record** for the preview subdomain pointing to your
> server's IP. For `https://123.preview.example.com`, create an A record for
> `*.preview.example.com` → server IP.

PRs opened **before** enabling preview deployments are not deployed
automatically — use **Load Pull Requests** to fetch open PRs, then deploy them
manually from the Preview Deployments page.

### Security: scoped deployments and secrets

Because a PR can run arbitrary code in your environment, control who can trigger
previews:

- **Preview Deployments** — only repository members, collaborators, and
  contributors can trigger PR deployments.
- **Allow Public PR Deployments** — anyone can trigger them. Enable with care.

Secrets are scoped: **Production** environment variables stay isolated to the
main deployment and are never exposed to previews; a separate set of **Preview
Deployment** variables is used only for PR previews (keep these non-sensitive so
contributors' PRs cannot reach production secrets).

### Setup methods

- **GitHub App** — when registering the app, enable the **Preview Deployments**
  option. For manual app setup (or to fix an app set up without it), grant
  `Read and write` on **Pull Requests** under Repository permissions and
  subscribe to the **Pull requests** event. Automated PR comments (deployment
  status, auto-updated) **only work with the GitHub App**.
- **Webhooks** — same secret/URL flow as push webhooks, but select **Let me
  select individual events** and choose **Pull Requests** instead of `push`.

## Health checks

Health checks let Coolify route traffic only to healthy instances — and are
**required for rolling updates** to work.

With Traefik as the reverse proxy:

- **Enabled** — traffic is routed only if the health check passes. A failing
  check causes `404 Not Found` or `No available server` errors (i.e. no traffic
  is routed to an unhealthy resource).
- **Disabled** — traffic is routed regardless of health status (unhealthy
  resources still receive traffic).

Enabling health checks for all resources is recommended.

### Configuring health checks

- **Applications** — two options:
  1. **UI** — set the path, expected response code, and interval. The container
     must have `curl` or `wget` installed to run the check.
  2. **Dockerfile** — use the `HEALTHCHECK` instruction.

  If both UI and Dockerfile health checks are enabled, **the Dockerfile takes
  precedence**.

- **Service stacks / Docker Compose build pack** — health checks must be defined
  in each service's `Dockerfile` or via the `healthcheck` attribute in the
  `docker-compose.yml`:

```yaml
services:
  web:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 10s
      timeout: 5s
      retries: 3
```

## Zero-downtime rolling updates

A rolling update starts a **new** container while the old one keeps serving.
Once the new container is confirmed **healthy**, the old one is stopped —
minimizing downtime.

All of these conditions must hold for rolling updates to work:

- **Health check** configured and passing — this is how Coolify confirms the new
  container is ready before stopping the old one. Without it the update cannot
  verify readiness.
- **Default container naming** — a custom container name prevents Coolify from
  managing instances correctly.
- **Not Docker Compose** — Compose deployments use static container names and are
  not supported for rolling updates.
- **No host port mapping** — if a port is mapped to the host, the new container
  cannot bind the same port while the old one runs, causing a conflict.

### Troubleshooting rolling updates

1. Verify the health check endpoint is correctly implemented and returns a valid
   response (e.g. HTTP 200); a failing/misconfigured check halts the update.
2. Confirm you are using the default container naming convention.
3. Check the `coolify-proxy` container or the application container logs for
   errors related to the update.

## Deployment webhook payloads

Coolify can POST JSON to a notification endpoint on deployment events (every
payload includes `success`, `event`, and `message`):

- `deployment_success` / `deployment_failed` — include `application_name`,
  `application_uuid`, `deployment_uuid`, `deployment_url`, `project`,
  `environment`, and `fqdn`. For **PR previews**, `fqdn` is omitted and
  `pull_request_id` + `preview_fqdn` are included instead.
- `status_changed` — sent when an application stops unexpectedly (not a manual
  stop).

Use these to drive external CI/CD steps or alerting.
