---
name: docker-networking
description: Connect Docker containers correctly — network drivers (bridge, host, overlay, none, macvlan/ipvlan), publishing ports, and Compose networking with service discovery by name across user-defined networks. Use when configuring container networking, port mappings, or compose networks.
---

# Docker networking

Containers have networking enabled by default and can make outgoing
connections. A container only sees a network interface, IP address, gateway,
routing table, and DNS — it has no knowledge of what kind of network it's on or
whether its peers are also containers. When Docker Engine first starts on Linux,
it has a single built-in `bridge` network (the "default bridge"); a container
run without `--network` is attached to it and gets internet access via
masquerading, no extra config needed.

## Rules of thumb

- **Use user-defined networks, not the default bridge.** On user-defined
  networks containers reach each other by **container/service name** (via
  Docker's embedded DNS at `127.0.0.11`); on the default bridge they can only
  reach each other by IP. User-defined networks also isolate groups of
  containers from each other.
- **Reference containers by name, never by IP.** IP addresses are assigned
  dynamically and change when a container is recreated; the name stays stable.
- **Publishing a port exposes it to the outside world by default.** Bind to
  `127.0.0.1` when a port should only be reachable from the Docker host.
- **Pick the driver for the job** — bridge for single-host container-to-
  container, host to drop isolation, overlay for multi-host/Swarm, macvlan/
  ipvlan for containers that must appear on the physical network, none to
  isolate completely.
- **Put backend services on an `--internal` network** so they have no external
  connectivity.

## Network drivers

| Driver | Use it when |
| --- | --- |
| `bridge` | Default. Containers on the same host need to talk to each other. User-defined bridge networks give an isolated network per project plus name-based discovery. |
| `host` | You want no network isolation between container and host; the container uses the host's networking directly. No port mapping. |
| `overlay` | Containers on **different** Docker hosts must communicate, or you're using Swarm services across nodes. Removes the need for OS-level routing. |
| `macvlan` | Containers must look like physical devices on your network, each with its own MAC address (e.g. migrating from VMs, legacy apps expecting direct L2). |
| `ipvlan` | Like macvlan but without unique MAC addresses — use when there's a limit on MAC addresses per interface/port; gives full control over IPv4/IPv6 and L2/L3. |
| `none` | Completely isolate a container from the host and other containers. Not available for Swarm services. |

Create a user-defined bridge network and attach a container:

```bash
docker network create -d bridge my-net
docker run --network=my-net -it busybox
```

A container can attach to **multiple networks** (pass `--network` repeatedly, or
use `docker network connect` on a running container) — e.g. a frontend on an
external bridge plus an `--internal` network to backend services. Docker picks
the default gateway and it may change as connections change; pin it with
`gw-priority` (default `0`; highest wins, so `gw-priority=1` makes a network the
default gateway).

You can also share another container's network stack directly with
`--network container:<name|id>`. In that mode flags like `--publish`,
`--hostname`, `--dns`, and `--add-host` are not supported.

## IP addressing and DNS

Each network allocates IPv4 by default (disable with `--ipv4=false`); enable
IPv6 with `--ipv6`. Subnets are either auto-allocated from
`default-address-pools` (configurable in `/etc/docker/daemon.json`) or set
explicitly:

```bash
docker network create --ipv6 --subnet 192.0.2.0/24 --subnet 2001:db8::/64 mynet
```

Containers on the default bridge get a copy of the host's `/etc/resolv.conf`;
containers on a user-defined network use Docker's embedded DNS server at
`127.0.0.11`, which forwards external lookups to the host's DNS servers. Per-
container DNS overrides: `--dns`, `--dns-search`, `--dns-opt`, `--hostname`, and
`--alias` (when connecting to an existing network).

## Publishing ports

By default the daemon blocks access to unpublished ports. Container ports on a
bridge network are reachable from the host and from other containers on the same
network, but not from outside the host or from other networks. Use `-p` /
`--publish` to expose a port, which sets up a NAT firewall rule:

```bash
docker run -p 8080:80 nginx              # host:8080 -> container:80 (TCP)
docker run -p 8080:80/udp nginx          # UDP
docker run -p 192.168.1.100:8080:80 ...  # bind to one host IP
```

**Publishing is insecure by default** — `-p 8080:80` makes the port reachable
from the outside world on all host addresses (`0.0.0.0` and `[::]`). To restrict
to the host only, bind to loopback:

```bash
docker run -p 127.0.0.1:8080:80 -p '[::1]:8080:80' nginx
```

> In releases older than 28.0.0, hosts on the same L2 segment could reach ports
> published to localhost (moby/moby#45610).

Other options:

- Change the default bind address for all user-defined bridge networks with the
  `com.docker.network.bridge.host_binding_ipv4` driver option (e.g.
  `127.0.0.1`), or per-daemon via `default-network-opts`.
- **Gateway modes** on the bridge driver
  (`com.docker.network.bridge.gateway_mode_ipv4`/`_ipv6`): `nat` (default,
  NAT + masquerading), `routed` (no NAT; only published ports open; container's
  own address used — for direct routing), `isolated` (only with `--internal`),
  and `nat-unprotected` (legacy, no port filtering — avoid).
- **Direct routing** avoids NAT (useful for IPv6); off by default. Enable with
  daemon `allow-direct-routing` or the
  `com.docker.network.bridge.trusted_host_interfaces` network option.
- Masquerading (NAT for outgoing packets) is on by default; disable per network
  with `com.docker.network.bridge.enable_ip_masquerade=false`.

## Networking in Compose

By default Compose creates one network named `<project-name>_default` (bridge
driver) and attaches every service to it. Each service registers its **name**
with the internal DNS, so services reach each other by service name with no IP
config. The project name comes from the directory name (override with
`--project-name` or `COMPOSE_PROJECT_NAME`).

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres:latest
    ports:
      - "8001:5432"
```

`web` connects to the database at `postgres://db:5432`. **Service-to-service
traffic uses the container port** (`5432`); the host port (`8001`) is only for
access from outside the network (`postgres://localhost:8001`). When a service
is recreated it rejoins under the same name with a new IP — always reference by
name, and have apps reconnect on dropped connections.

### Custom and isolated networks

Define your own networks under the top-level `networks` key and attach services
selectively to build topologies. Services that share no network can't reach each
other:

```yaml
services:
  proxy:
    build: ./proxy
    networks: [frontend]
  app:
    build: ./app
    networks: [frontend, backend]
  db:
    image: postgres:latest
    networks: [backend]

networks:
  frontend:
    driver: bridge
  backend: {}
```

- `internal: true` creates a network with no host connectivity / external
  gateway — ideal for databases. A service on both an internal and a
  non-internal network can still reach the internet via the non-internal one.
- Override the default network's settings with a `networks: default:` entry.
- `network_mode` overrides per service: `host` (shares host stack — no port
  mapping, **no service-name DNS**; use only when genuinely required),
  `none` (no networking), `service:{name}`, `container:{name}`.

### External networks (multiple projects)

To connect services across separate Compose projects, create a shared network
once and mark it `external` in each project. The network **must exist before**
`docker compose up`, or Compose fails with `Network not found`:

```bash
docker network create inter-project
```

```yaml
services:
  api:
    image: myapi:latest
    networks: [shared, default]   # also keep the project-internal network
networks:
  shared:
    external: true
    name: inter-project
```

Services on the same external network reach each other by service name. Combine
an external network with a project-internal one (hybrid networking) to expose
only the services that need to be reachable while keeping databases isolated.

### Debugging connectivity

Work in order — config, then membership, then live connectivity:

```bash
docker compose port db 5432              # which host port maps to container 5432
docker compose port --index=2 web 80     # a specific replica's dynamic port
docker network inspect <network-name>    # which containers are attached
docker compose exec <svc> cat /etc/hosts # inspect injected hosts
```

`extra_hosts` adds custom hostname→IP entries (including
`host.docker.internal:host-gateway` to reach the host).
