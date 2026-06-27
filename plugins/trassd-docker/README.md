# trassd-docker

Skills and agents that enforce **Docker** best practices: writing efficient,
secure Dockerfiles, multi-stage builds, build-cache optimization, build secrets
& args, multi-platform builds, building in CI (GitHub Actions), Compose file
authoring, Compose environment & secrets, container/image security hardening,
networking, and persistent storage with volumes.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `dockerfile-best-practices` | Cache-friendly instruction order, minimal layers, COPY vs ADD, pinned base images, non-root USER, ENTRYPOINT/CMD, `.dockerignore` |
| `docker-multi-stage-builds` | Build vs runtime stages, `COPY --from`, named/target stages, slim production images |
| `docker-build-cache` | Layer caching & invalidation, instruction ordering, `RUN --mount=type=cache`, registry/GHA cache backends |
| `docker-build-secrets-args` | `RUN --mount=type=secret`, build `ARG` vs runtime `ENV`, keeping secrets out of layers |
| `docker-multi-platform` | `--platform` multi-arch builds, buildx builders & drivers, emulation vs native, cross-compilation |
| `docker-build-ci` | GitHub Actions build-push-action, cache, multi-platform matrix, tags/labels, secrets, test-before-push |
| `docker-compose-best-practices` | services/networks/volumes model, `depends_on` health conditions, profiles, restart, production |
| `docker-compose-env-secrets` | `env_file`, interpolation & precedence, the `secrets` mechanism |
| `docker-container-security` | Non-root, userns-remap, seccomp/AppArmor, capabilities, CPU/memory limits |
| `docker-networking` | Network drivers (bridge/host/overlay/macvlan), port publishing, Compose service discovery |
| `docker-storage-volumes` | Named volumes vs bind mounts vs tmpfs, persistence & performance trade-offs |

## Agents

| Agent | When to use |
|-------|-------------|
| `dockerfile-reviewer` | Review a Dockerfile against Docker's official best practices. |
| `docker-compose-auditor` | Audit a `compose.yaml` for correctness and security. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-docker@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
