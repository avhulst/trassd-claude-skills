---
name: coolify-server-hardening
description: >-
  Set up and harden the Linux servers Coolify manages — connecting localhost and
  remote servers over SSH, configuring OpenSSH, optionally running a non-root
  deploy user, opening the right firewall ports, patching OS packages, and
  scheduling automated Docker cleanup. Use when connecting or securing a server,
  fixing SSH/firewall access, managing multiple servers, or preventing disk
  exhaustion from Docker.
---

# Coolify Server Hardening

Coolify connects to every server it manages — including the `localhost` server
where Coolify itself runs — over **SSH with key authentication**. Requirements
for any server:

- SSH connectivity with **SSH key authentication** (public key in the
  managing user's `~/.ssh/authorized_keys`).
- **Docker Engine 24+**.

Server types: **Localhost** (where Coolify is installed; usable for resources
but not recommended since high load can starve Coolify) and **Remote Server**
(any remote Linux box — VPS, Raspberry Pi, laptop).

## SSH and OpenSSH setup

The SSH key used for the Coolify connection **must not have a passphrase or 2FA**
or the connection fails. By default the public key goes in the **root** user's
`~/.ssh/authorized_keys`.

Install and enable OpenSSH (Debian/Ubuntu shown; other distros use `sshd`):

```bash
apt update && apt install -y openssh-server
systemctl enable --now ssh
```

Recommended `/etc/ssh/sshd_config` settings, then restart SSH:

```bash
PubkeyAuthentication yes
PermitRootLogin prohibit-password   # recommended over `yes`/`without-password`
```

```bash
systemctl restart ssh   # use `systemctl restart sshd` on RHEL/Arch/openSUSE,
                        # `rc-service sshd restart` on Alpine
```

IMPORTANT: add your public key to `~/.ssh/authorized_keys` **before** setting
`PermitRootLogin prohibit-password`, or you can lock yourself out.

The Coolify install script normally handles SSH config automatically; do the
above manually only if that fails. If generating the key on the server yourself,
Coolify expects it at `/data/coolify/ssh/keys/id.root@host.docker.internal`,
owned by UID `9999`:

```bash
ssh-keygen -t ed25519 -a 100 \
  -f /data/coolify/ssh/keys/id.root@host.docker.internal -q -N "" -C root@coolify
chown 9999 /data/coolify/ssh/keys/id.root@host.docker.internal
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

Paste the private key into Coolify (**Security → Private keys**), attach it to
the server, then **Validate Server & Install Docker Engine** on the server's
General page — a green **Proxy Running** status confirms success.

**Cloudflare Tunnels:** Coolify does **not** install `cloudflared`; install it
first. Coolify only adds the SSH `ProxyCommand` for tunneled connections.

## Non-root deploy user (experimental)

You can have a non-root user manage resources. Requirements: its SSH key is on
the server, and it has passwordless sudo. Add to `/etc/sudoers`:

```bash
your-non-root-user ALL=(ALL) NOPASSWD: ALL
```

Note: this grants full passwordless root and is **not** the most secure setup —
more granular per-binary permissions are planned.

## Firewall ports

Docker uses NAT-based iptables rules that **bypass UFW**, so blocking ports with
UFW alone does not work. **Prefer your cloud provider's firewall** to manage open
ports. If the provider has none, the community tool
[`ufw-docker`](https://github.com/chaifeng/ufw-docker) bridges UFW and Docker.

Ports to open:

| Port | Purpose | Self-hosted | Coolify Cloud |
|------|---------|:-----------:|:-------------:|
| 22 (or custom) | SSH access | yes | yes |
| 80  | SSL cert generation via proxy | yes | yes |
| 443 | HTTPS traffic | yes | yes |
| 8000 | HTTP to Coolify dashboard | yes | — |
| 6001 | Real-time communication | yes | — |
| 6002 | Terminal access (v4.0.0-beta.336+) | yes | — |

Self-hosted with a custom domain behind the integrated proxy: you may close
**8000**, **6001**, and **6002** after first dashboard access. For Coolify Cloud
only SSH (22) is needed for management; restrict it by IP using Coolify Cloud's
published [IPv4](https://coolify.io/ipv4.txt)/[IPv6](https://coolify.io/ipv6.txt)
lists if desired. GitHub webhooks require **80 and 443** open. On Oracle Cloud
Free ARM, also open the ports in Oracle's dashboard.

CAUTION: bad firewall changes can lock you out and be hard to recover — proceed
only if necessary and understood.

## OS patching

**Server Patching** (v4.0.0-beta.419+) updates OS packages from the dashboard:
**Servers → [Server] → Security tab → Server Patching**. Update packages
individually or use **Update All Packages**.

- Supported package managers: **APT**, **DNF**, **Zypper**.
- Coolify **never auto-installs** updates — it only checks and lists them;
  updates apply only when you click Update.
- Coolify checks weekly and sends notifications (enabled by default; managed in
  Notification Settings). The check interval is fixed, but **Check now** forces
  one.
- WARNING: some updates can break features; **Docker updates restart Docker**,
  taking all apps and Coolify itself offline until it restarts. Review each
  package before updating.

## Automated Docker cleanup

Prevents servers running out of disk. Configure under **Servers → [Server] →
Configuration → Advanced**:

- **Docker Cleanup Threshold** — disk-usage percentage that triggers cleanup
  (e.g. 80%).
- **Docker Cleanup Frequency** — [cron expression](https://coolify.io/docs/knowledge-base/cron-syntax)
  schedule, used when **Force Docker Cleanup** is enabled.
- **Optional cleanups** — unused volumes (can cause **data loss**) and unused
  networks.

Recommended: enable **Force Docker Cleanup** with a cron schedule — more
reliable than relying on the disk threshold.

When triggered, cleanup removes stopped Coolify-managed containers (no data
loss), unused images, Docker build cache, old Coolify helper-image versions, and
optionally unused volumes/networks. It only affects Coolify-managed resources
and **skips while a deployment is in progress** to avoid deleting in-use images.

## Multiple-server note

Each server runs its **own proxy** and serves its own apps directly — traffic to
apps on secondary servers does **not** route through the main Coolify server.
Point each app's domain DNS at the **server where that app is deployed**, not at
the main Coolify server. The main server only provides the management UI, SSH
deployments, health checks, and monitoring.
