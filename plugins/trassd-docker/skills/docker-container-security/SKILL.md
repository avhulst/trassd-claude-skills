---
name: docker-container-security
description: Harden Docker containers and images at runtime — run as non-root, user-namespace remapping, seccomp and AppArmor profiles, dropped Linux capabilities, and CPU/memory resource constraints. Use when reviewing or configuring container/image security and isolation.
---

# Docker container security

Containers are isolated through kernel namespaces (process, network, mount, …)
and constrained through control groups (cgroups). Namespaces stop a container
from seeing or affecting processes in other containers or the host; cgroups
account for and limit memory, CPU, and disk I/O so a single container can't
exhaust a resource and bring the host down. These are the foundation — the
practices below layer extra hardening on top.

## Rules of thumb

- **Run application processes as an unprivileged user inside the container.**
  This is the single most effective defense against privilege-escalation
  attacks from within a container. Containers are quite secure by default,
  *especially* when their processes run as non-privileged users.
- **Drop all capabilities, then add back only what the process needs.** Docker
  already drops all but a needed allowlist by default. Prefer removing
  capabilities over adding them — adding makes the container *less* secure.
- **Keep the default seccomp and AppArmor profiles.** Both are moderately
  protective with wide compatibility; don't disable them without a clear reason.
- **Set memory and CPU limits** so one container can't destabilize the host.
- **Don't run `--privileged`** unless genuinely required, and only in trusted
  environments — it grants elevated permissions and exposes the host.
- **Only trusted users should control the Docker daemon.** The daemon needs
  root (unless using rootless mode), and features like host bind mounts let a
  container alter the host filesystem without restriction.

## Run as non-root

The best way to prevent in-container privilege escalation is to configure the
application to run as an unprivileged user. Most server tasks that traditionally
needed root (SSH, cron, logging, hardware/network management) are handled by the
infrastructure around the container, not inside it — so containers rarely need
real root.

Capabilities turn the root/non-root split into fine-grained access control. For
example, a process that only needs to bind a port below 1024 can be granted
`CAP_NET_BIND_SERVICE` instead of running as root.

## User-namespace remapping

For containers whose process *must* run as `root` inside the container, remap
that root to a less-privileged user on the host. The mapped user gets a range of
UIDs that look like UIDs 0–65536 inside the namespace but have no privileges on
the host. If a process escapes the namespace, it runs as an unprivileged
high-number UID that doesn't even map to a real host user.

Remapping is driven by `/etc/subuid` and `/etc/subgid` (one for UIDs, one for
GIDs). An entry like `testuser:231072:65536` means host UID `231072` appears as
UID `0` inside the namespace, through `231072 + 65536 - 1`. Ranges must never
overlap, or a process could reach another namespace.

Enable it on the daemon via `/etc/docker/daemon.json` (recommended over the
`--userns-remap` flag), then restart Docker:

```json
{
  "userns-remap": "testuser"
}
```

Use `"default"` to have Docker create and use a `dockremap` user automatically.

Important constraints:

- `userns-remap` keeps the *daemon* running as root. To run the daemon and
  containers without root entirely, use rootless mode instead.
- Enable it on a **new** install. Enabling masks existing images, containers,
  and other objects under `/var/lib/docker/` (they're moved into a
  UID/GID-named subdirectory); disabling it later loses access to anything
  created while it was on.
- Incompatible features: sharing the host PID or NET namespace
  (`--pid=host`, `--network=host`), `--privileged` without also passing
  `--userns=host`, and volume/storage drivers unaware of daemon user mappings.
- The kernel restricts a user-namespaced root, e.g. `mknod` (device creation)
  is denied.
- Disable remapping for a single container with `--userns=host` on
  `docker container create/run/exec` (note the filesystem-ownership side effect
  documented for that flag).

## Seccomp profiles

Seccomp (secure computing mode) restricts which syscalls a container can make.
Available only when Docker was built with seccomp and the kernel has
`CONFIG_SECCOMP` enabled (`grep CONFIG_SECCOMP= /boot/config-$(uname -r)`).

The default profile is an allowlist: it sets `defaultAction` to
`SCMP_ACT_ERRNO` (deny with Permission Denied) and explicitly allows the needed
syscalls. It disables around 44 of 300+ syscalls — moderately protective with
wide compatibility. It is **instrumental for least-privilege** and **should not
be changed.**

Blocked syscalls include dangerous operations like `mount`/`umount`, `ptrace`,
`reboot`, `kexec_load`, kernel-module calls (`init_module`, `delete_module`),
keyring calls (`add_key`, `keyctl`), and the `io_uring_*` family (container
breakout risk). Most are also gated behind capabilities such as
`CAP_SYS_ADMIN`, `CAP_SYS_MODULE`, or `CAP_SYS_BOOT`.

Override only if you must:

```bash
docker run --rm -it --security-opt seccomp=/path/to/profile.json hello-world
```

Passing `--security-opt seccomp=unconfined` runs without any profile — avoid in
production.

## AppArmor profiles

On AppArmor systems, Docker auto-generates and loads a `docker-default` profile
that runs on **containers** (not the daemon). It is moderately protective with
wide compatibility and is applied unless overridden.

Run the default explicitly, or a custom profile:

```bash
docker run --rm -it --security-opt apparmor=docker-default hello-world
docker run --security-opt apparmor=your_profile nginx
```

Load and unload custom profiles with `apparmor_parser`:

```bash
apparmor_parser -r -W /etc/apparmor.d/containers/docker-nginx   # load
apparmor_parser -R /etc/apparmor.d/containers/docker-nginx      # unload
```

Profiles can `deny network raw` (blocks raw sockets / ping), deny writes to
sensitive paths, and restrict capabilities. Debug with `dmesg` (look for
`apparmor="DENIED"` lines) and check loaded/enforced profiles with `aa-status`
— `docker-default` should be in `enforce` mode on container PIDs. A worked Nginx
example profile is in [references/apparmor-nginx-profile.md](references/apparmor-nginx-profile.md).

## CPU and memory resource constraints

By default a container has no resource limits and can use as much memory/CPU as
the kernel scheduler allows. On Linux, if memory runs out the kernel throws an
OOM exception and starts killing processes — potentially the wrong ones. Docker
biases the OOM killer to spare the daemon, but containers aren't protected.
Mitigate by testing your app's memory needs and setting limits.

### Memory

| Flag | Effect |
| --- | --- |
| `-m` / `--memory` | Hard limit on memory the container can use. Minimum `6m`. |
| `--memory-swap` | Total memory + swap. Modifier on `--memory`; see below. |
| `--memory-reservation` | Soft limit (must be lower than `--memory`); kicks in under host contention. |
| `--memory-swappiness` | 0–100, tunes anonymous-page swapping. `0` disables it. |
| `--oom-kill-disable` | Disable the OOM killer — only ever with `-m` set. |

`--memory-swap` only has meaning when `--memory` is set. To **prevent swap
entirely, set `--memory-swap` equal to `--memory`.** With `--memory="300m"` and
`--memory-swap="1g"`, the container gets 300m RAM + 700m swap. `-1` means
unlimited swap. Inside the container, `free` reports the *host's* swap, not the
container's — don't trust it.

Don't try to circumvent the OOM safeguards by setting an extreme negative
`--oom-score-adj` or `--oom-kill-disable` on the daemon or a container.

### CPU

Most users use the default CFS scheduler. `--cpus` is the convenient option:

```bash
docker run -it --cpus=".5" ubuntu /bin/bash
```

`--cpus="1.5"` guarantees at most 1.5 CPUs — equivalent to
`--cpu-period=100000 --cpu-quota=150000`.

| Flag | Effect |
| --- | --- |
| `--cpus=<value>` | How much CPU the container may use (e.g. `1.5`). Preferred. |
| `--cpu-period` / `--cpu-quota` | Lower-level CFS controls; `--cpus` is usually better. |
| `--cpuset-cpus` | Pin to specific cores, e.g. `0-3` or `1,3`. |
| `--cpu-shares` | Relative weight (default 1024); a soft priority, only enforced under contention. |

The real-time scheduler (`--cpu-rt-runtime`, `--ulimit rtprio`,
`--cap-add=sys_nice`) is an advanced kernel feature most users never need;
misconfiguring it can make the host unstable.

## Related kernel features

Docker enables capabilities by default but doesn't interfere with other
hardening: AppArmor, SELinux, GRSEC/PAX, TOMOYO can all run alongside it
system-wide. Docker Content Trust can be configured in `daemon.json`
(trustpinning) so the daemon only runs signed images.
