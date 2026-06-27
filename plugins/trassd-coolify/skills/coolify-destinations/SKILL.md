---
name: coolify-destinations
description: >-
  Manage where Coolify runs containers — Docker (Standalone/Swarm) destinations,
  default vs. custom Docker networks, network isolation, and connecting a
  destination to a server. Use when adding or deleting a destination, isolating
  resources on a custom Docker network, defining a custom Docker address pool at
  install time, connecting a service stack to a predefined network, or deploying
  to a specific Docker engine or Swarm cluster.
---

# Coolify Destinations

A **destination** in Coolify is a [Docker network](https://docs.docker.com/engine/network/)
on a server that acts as the deployment target for applications, databases, and
services. Deploying to a destination gives resources network isolation and
organization.

## Core model

- A destination **belongs to exactly one server**; a server can have **multiple**
  destinations. Destinations are tied to that server's Docker daemon.
- Resources on **different** destinations cannot communicate unless explicitly
  configured — network isolation is the default benefit.
- The same application can be deployed to multiple destinations/servers.

## Destination types (determined by the server, not chosen)

The type is set automatically by how the server was configured when added to
Coolify — you cannot pick it manually when creating the destination.

- **Standalone Docker** — single-server deployments. Uses a standard Docker
  bridge or custom network.
- **Docker Swarm** — multi-node cluster deployments. Uses overlay networks for
  cross-node communication. Requires Swarm mode enabled during server setup.
  (Swarm is an experimental feature.)

## Creating a destination

Two entry points, then fill in the details:

- **Destinations** in the main nav → **+ Add**.
- **Servers** → select server → **Destinations** tab → **+ Add**.

Configuration fields:

- **Destination Name** — display name; auto-generated from server name + network
  ID, editable.
- **Network Name** — the actual Docker network name. Auto-generated (CUID2),
  must be unique per server, editable at creation but **cannot be changed
  afterward**.
- **Server** — must be online and accessible. **Cannot be a build server.**

On create, Coolify automatically creates the Docker network on the server,
connects the proxy (Traefik/Caddy) to it, configures isolation, and enables
inter-container communication within the network.

### Importing existing networks

Use **Server → Destinations → Scan for Destinations** to detect Docker networks
already on the host and import the selected ones as destinations.

Common error: **"Network already added to this server"** — the network name
conflicts with an existing destination.

## Managing and deleting

View all destinations under **Destinations**, or per-server under
**Servers → [Server] → Destinations**. Click a destination to edit its display
**Name**; the **Server IP** and **Docker Network** name are read-only.

Coolify **refuses to delete a destination that has active resources**. Before
deleting, confirm no applications, databases, or services use it, and check that
nothing depends on it (hardcoded env var references, cross-destination network
dependencies, proxy/load-balancer config).

### Assigning resources

- When a server has more than one destination, you are prompted to pick one when
  creating a resource.
- To put an existing resource on another destination, **Clone** it via the
  resource's **Resource Operations** page and select the target destination.
  Cloning creates a duplicate (new instance) — it does **not** move the resource
  or its data.

### Service stacks and predefined networks

Service stacks — and applications using the **Docker Compose Build Pack** — are
**not** connected to the assigned destination by default. Coolify gives each
service stack its own isolated network so you can run multiple instances of the
same service on one server without conflicts.

To let a service stack talk to other resources on the same destination, enable
**Connect to Predefined Networks** in its settings.

Do **not** define network configuration directly in a service stack's
`docker-compose.yml`/`docker-compose.yaml` — use Coolify's destination settings
instead. Manual network blocks can cause undesired behavior such as Gateway
Timeout errors.

## Custom Docker network (address pool) at install time

To control the CIDR range Docker hands out to networks, set environment
variables when running the Coolify install script. Requirements:

- `DOCKER_ADDRESS_POOL_BASE` — a valid CIDR block, e.g. `10.0.0.0/8`.
- `DOCKER_ADDRESS_POOL_SIZE` — a number, e.g. `10`.
- `DOCKER_POOL_FORCE_OVERRIDE` — only needed to override an address pool that
  already exists on the host.

Automated install (run as `root`; the script falls back to `sudo` if not):

```bash
env DOCKER_ADDRESS_POOL_BASE=10.0.0.0/8 DOCKER_ADDRESS_POOL_SIZE=10 \
  bash -c 'curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash'
```

Manual install — add the variables to `/data/coolify/source/.env`:

```bash
DOCKER_ADDRESS_POOL_BASE=10.0.0.0/8
DOCKER_ADDRESS_POOL_SIZE=10
DOCKER_POOL_FORCE_OVERRIDE=false
```

## Docker Swarm destinations

Swarm is experimental. To deploy to a Swarm, add the **Swarm Manager** to
Coolify; optionally add **Swarm Workers** so Coolify can run cleanups on them.

Key requirements:

- An **external Docker registry** is mandatory — the manager pushes built images
  and every worker pulls them. Set Docker login credentials accordingly.
- Persistent-storage services need a **shared volume** across workers (NFS, AWS
  EFS, GlusterFS, etc.) so the Swarm can move the service between workers.

Minimal cluster bring-up (at least 3 same-architecture servers: 1 manager,
2+ workers):

```bash
# On the manager — use the private IP if you have private networking
docker swarm init --advertise-addr <MANAGER_IP>
# Run the printed `docker swarm join --token ...` command on each worker
docker node ls   # verify: manager shows Leader, workers Ready/Active
```

On Hetzner, set the overlay MTU to 1450 in `/etc/docker/daemon.json` under
`default-network-opts.overlay` and restart Docker, or networking will misbehave.
