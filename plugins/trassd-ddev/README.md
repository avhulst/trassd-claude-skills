# trassd-ddev

Skills and agents for **DDEV** — the Docker-based local development environment
used by PHP/CMS projects. Covers project setup & configuration, custom services
& Docker Compose overrides, custom commands & hooks, debugging & profiling,
add-ons, and CMS-specific settings.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `ddev-project-setup` | `ddev config`, `.ddev/config.yaml` (type/docroot/versions), project lifecycle |
| `ddev-custom-services` | Additional services, `docker-compose.*.yaml` overrides, image customization, hostnames |
| `ddev-custom-commands-hooks` | Custom commands (`.ddev/commands`), config.yaml hooks |
| `ddev-debugging-profiling` | Xdebug step debugging; Blackfire / Xdebug / XHProf profiling |
| `ddev-addons` | Using (`ddev add-on get`) and authoring add-ons (install.yaml) |
| `ddev-cms-settings` | Auto-generated CMS settings, DB credentials, import/export/snapshot |

## Agents

| Agent | When to use |
|-------|-------------|
| `ddev-config-reviewer` | Review `.ddev` config, compose overrides, commands & hooks against DDEV best practices. |
| `ddev-troubleshooter` | Diagnose startup/networking/port/provider failures and misconfiguration. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-ddev@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
