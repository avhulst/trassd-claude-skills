---
name: coolify-persistent-storage
description: >-
  Persist data across deployments with Coolify persistent storage — Docker
  volumes vs bind mounts (file/directory from the host), destination path
  mapping, and avoiding data loss on redeploy. Use when a resource needs data to
  survive restarts/redeploys or when configuring storage mounts.
---

# Coolify Persistent Storage

Add persistent storage to a resource so its data survives between deployments.
Anything written outside a persistent mount is lost when the container is
redeployed. The available storage types depend on the destination; on a Docker
Engine destination you can use a **volume** or a **bind mount**.

## Container base directory

The base directory inside the container is `/app`. To store files under a
`storage` directory, set the destination path to `/app/storage` — not
`storage` or `/storage`. This applies to both volumes and bind mounts.

## Volume

A Docker-managed volume. Define:

- **Name** of the volume.
- **Destination Path** — where it mounts inside the container (e.g. `/app/storage`).

To prevent storage overlap between resources, Coolify automatically prepends the
resource's UUID to the volume name, so each resource gets its own volume.

Use a volume when you want Docker to manage the storage and don't need the data
at a specific location on the host filesystem.

## Bind mount

Mounts a file or directory from the host (your server) into the container. **No
Docker volume is created.** Define:

- **Name** — used only as a reference.
- **Source Path** — the path on the host system.
- **Destination Path** — where it mounts inside the container (e.g. `/app/storage`).

Use a bind mount when the data must live at a known location on the host (for
inspection, backups, or sharing with host tooling).

## Avoiding data loss and pitfalls

- Mount any directory your app writes to (uploads, databases, generated files)
  to a volume or bind mount; otherwise it is wiped on each redeploy.
- Get the **destination path** right relative to `/app` — a path like `storage`
  rather than `/app/storage` will not map where your app expects.
- **Sharing one file across multiple containers is not recommended.** If you
  mount the same file into more than one container, you must ensure your
  resources implement proper file locking.
