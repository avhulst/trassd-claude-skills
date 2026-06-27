# trassd-deployer

Skills and agents that enforce **Deployer** ([deployer.org](https://deployer.org))
best practices: writing `deploy.php` / `deploy.yaml` recipes, defining hosts &
connections, tasks & hooks, the release/shared/current deploy lifecycle,
framework recipes, server provisioning, CI/CD pipelines, and zero-downtime
PHP-FPM/opcache configuration.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `deployer-recipe-structure` | `namespace Deployer`, recipe imports, `set()` config, PHP vs YAML vs MAML, `dep init` |
| `deployer-hosts` | `host()`, hostname/alias, remote_user, deploy_path, host ranges, labels & selectors, YAML inventory |
| `deployer-tasks-hooks` | `task()`/`desc()`/`run()`/`{{…}}`, `before`/`after` hooks, task groups, `once`/`limit`/`select` |
| `deployer-deploy-lifecycle` | releases/current/shared layout, prepare/publish, shared & writable, keep_releases, atomic symlink, rollback |
| `deployer-framework-recipes` | `require 'recipe/<fw>.php'`, overriding shared/writable defaults, framework tasks, bin/console |
| `deployer-provisioning` | `dep provision` — PHP/database/Node/website/user/firewall, Caddy public_path |
| `deployer-cicd` | GitHub Actions / GitLab CI / Bitbucket, concurrency guards, SSH & dotenv secrets |
| `deployer-zero-downtime` | PHP-FPM/opcache symlink fix (`$realpath_root` / `resolve_root_symlink` / `revalidate_path`), cachetool |

## Agents

| Agent | When to use |
|-------|-------------|
| `deployer-recipe-reviewer` | Review a `deploy.php`/`deploy.yaml`/`deploy.maml` recipe against Deployer best practices. |
| `deployer-deployment-auditor` | Audit a deployment setup and its CI/CD pipeline for safety and security. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-deployer@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
