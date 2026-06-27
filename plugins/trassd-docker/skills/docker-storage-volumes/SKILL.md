---
name: docker-storage-volumes
description: Persist and share Docker container data correctly — choosing between named volumes, bind mounts, and tmpfs mounts, with their persistence and performance trade-offs. Use when configuring storage for containers or compose services.
---

# Docker storage and volumes

By default, files a container writes go to its **writable layer**, which sits on
top of the read-only image layers and is destroyed when the container is
removed. Getting data out of the writable layer is hard, and writing to it is
slower than writing to the host because it goes through a storage driver's union
filesystem. To persist or share data, store it outside the writable layer using
one of three mount types: volumes, bind mounts, or tmpfs.

## Rules of thumb

- **Volumes are the preferred mechanism for persisting data.** They're managed
  by Docker, isolated from the host, easy to back up/migrate, work on Linux and
  Windows, can be shared safely between containers, and give the same raw
  performance as writing to the host filesystem.
- **Use bind mounts only when you need files accessible from both the host and
  the container** — typically sharing source code or build artifacts during
  development. They tie the container to the host's directory layout.
- **Use tmpfs for non-persistent, in-memory data** — caches, scratch state, or
  sensitive data you don't want written to disk (Linux only).
- **Don't write app data straight into the container's writable layer** — it
  doesn't persist, can't be extracted easily, grows the container, and is slower.
- **Mount read-only (`ro`) when the container only needs to read** the data.
- Mounting **over a directory that already has files obscures** those files
  (like mounting a USB drive over `/mnt`); there's no clean way to un-obscure
  short of recreating the container.

## Choosing a mount type

| | Volume | Bind mount | tmpfs |
| --- | --- | --- | --- |
| Managed by | Docker (`/var/lib/docker/volumes/`) | You (any host path) | Host RAM |
| Persists after container removal | Yes | Yes (on host) | No |
| Accessible from host | No (don't touch directly) | Yes | No |
| Shareable between containers | Yes | Yes | No |
| Platform | Linux + Windows | Linux + Windows | Linux only |
| Best for | Databases, app data, high-performance I/O | Source code, build artifacts, host config files | Caches, secrets, scratch state |

All three look identical from inside the container — a directory or file in its
filesystem. The `--mount` flag is generally preferred over `-v`/`--volume`
because it's more explicit and supports all options; `--mount` is required for
volume driver options, volume subpaths, and Swarm services.

## Volumes

Volumes are persistent stores created and managed by Docker. Their contents
outlive any single container, and a volume can be mounted into multiple
containers at once. **Don't access the on-host volume data directly** — it's
unsupported and may break the volume.

```bash
docker volume create my-vol
docker volume ls
docker volume inspect my-vol
docker volume rm my-vol
docker volume prune          # remove all unused volumes
```

Mount into a container (these are equivalent; Docker creates the volume if it
doesn't exist):

```bash
docker run -d --name devtest --mount source=myvol2,target=/app nginx:latest
docker run -d --name devtest -v myvol2:/app nginx:latest
```

Behaviors to know:

- **Named vs anonymous.** Named volumes you reference by name. Anonymous volumes
  get a random name and also persist — *except* when the container was started
  with `--rm`, which removes the anonymous volume on exit. Anonymous volumes
  aren't shared/reused automatically.
- **Pre-population.** Mounting an *empty* volume into a container directory that
  already contains files copies those files into the volume (good for seeding
  data for other containers). Disable with the `volume-nocopy` option. A
  *non-empty* volume obscures the directory's existing contents instead.
- **Read-only:** add `ro`/`readonly`. The same volume can be read-write for one
  container and read-only for another.
- **Subpath:** `--mount ...,volume-subpath=app1` mounts just a subdirectory
  (must already exist in the volume) — useful for giving each container its own
  subdirectory of a shared volume.
- **Volume removal is always a separate step** — removing a container or a
  service does not remove its volumes.

In Compose, declare volumes under the top-level `volumes` key:

```yaml
services:
  frontend:
    image: node:lts
    volumes:
      - myapp:/home/node/app
volumes:
  myapp:
```

Add `external: true` under the volume to reference a volume created outside
Compose. Volume **drivers** (`-d`/`volume-driver` with `volume-opt`) let you
back a volume with external storage like NFS, CIFS/Samba, or SFTP — useful for
sharing files across machines (drivers requiring options must use `--mount`).
Back up by mounting the volume into a helper container with `--volumes-from` and
`tar`-ing its contents to a bind-mounted host directory.

## Bind mounts

A bind mount links an existing host path directly into the container. Unlike
volumes, both host (including non-Docker processes) and container can modify the
files simultaneously, and there's no Docker isolation.

```bash
docker run --mount type=bind,src="$(pwd)"/target,dst=/app nginx:latest
docker run -v "$(pwd)"/target:/app nginx:latest
```

Considerations:

- **Write access to the host by default** — a container can create, modify, or
  delete host files, with security implications. Use `ro`/`readonly` to prevent
  writes.
- **Bind mounts resolve on the daemon host**, not the client — you can't bind
  mount client files against a remote daemon. (Docker Desktop transparently
  bridges native host paths into its Linux VM.)
- **Ties the container to the host's directory structure** — it may fail on a
  host that lacks the expected path.
- **Source must exist.** `-v` auto-creates a missing source as a directory;
  `--mount` errors instead unless you pass `bind-create-src`.
- **SELinux:** add `z` (shared) or `Z` (private) to relabel — use with extreme
  caution; relabeling a system dir like `/home` or `/usr` can break the host.
  (Ignored for services.)
- **Bind propagation** (`rprivate` default, plus `shared`/`slave`/`private` and
  recursive `r*` variants) and **recursive mounts** (`bind-recursive`) are
  advanced, Linux-only, and most users never configure them.

In Compose, use the long form:

```yaml
services:
  frontend:
    image: node:lts
    volumes:
      - type: bind
        source: ./static
        target: /opt/app/static
```

## tmpfs mounts

A tmpfs mount stores files in the **host's memory**, never on disk. It's
temporary: removed when the container stops or restarts, lost on host reboot.
Linux only.

```bash
docker run -d --name tmptest --mount type=tmpfs,destination=/app nginx:latest
docker run -d --name tmptest --tmpfs /app nginx:latest
```

Use it for data that must not persist — non-persistent state (to protect
performance and avoid the writable layer) or sensitive data like credentials.
There is no `source` for tmpfs mounts.

Limitations and notes:

- **Can't be shared between containers**; Linux only.
- Because it maps to kernel `tmpfs`, data **may still be written to a swap
  file** and thus reach disk.
- Setting permissions on tmpfs may reset after a container restart (setting
  `uid`/`gid` can work around it).
- `--mount` options: `tmpfs-size` (bytes; default max is 50% of host RAM) and
  `tmpfs-mode` (octal, default `1777`). `--tmpfs` supports more mount options
  (`ro`, `noexec`, `nosuid`, `size`, `mode`, …) but can't be used with Swarm
  services — use `--mount` there.

```bash
docker run --mount type=tmpfs,dst=/app,tmpfs-size=21474836480,tmpfs-mode=1770 nginx:latest
docker run --tmpfs /data:noexec,size=1024,mode=1777 nginx:latest
```
