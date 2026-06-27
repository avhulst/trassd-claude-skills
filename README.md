# Trassd

Claude Code plugins that enforce **PHP-framework best practices** (Symfony,
Shopware, Contao, Sulu, DDEV, …) to raise code quality in downstream projects.
Each plugin bundles **skills** and **agents** distilled from the framework's
official documentation.

This repository is a single Claude Code **marketplace** named `trassd` that
catalogs all plugins below.

## Installation

Add the marketplace, then install the plugins you need:

```
/plugin marketplace add <git-url-of-this-repo>
/plugin install trassd-symfony@trassd
/plugin install trassd-shopware@trassd
```

The install id is `<plugin-name>@trassd` (plugin name `@` marketplace name).

## Plugins

| Plugin | Skills | Agents | Focus |
| --- | --- | --- | --- |
| `trassd-contao` | 16 | 2 | Contao CMS extension/bundle best practices — DCA, fragment & page controllers, Twig, migrations, security voters, QA stack. |
| `trassd-ddev` | 6 | 2 | DDEV local dev environment — setup & config, custom services, commands & hooks, debugging, add-ons. |
| `trassd-shopware` | 15 | 3 | Shopware 6 plugin & app best practices — DI, DAL entities, migrations, Store-API, Storefront, Administration, apps, themes. |
| `trassd-shopware-frontends` | 6 | 1 | Shopware Frontends (Vue 3 / Nuxt 4) — project setup, API client, composables, CMS base layer, type generation. |
| `trassd-sulu` | 8 | 2 | Sulu CMS best practices — templates, content architecture, webspaces, admin UI, rendering, security, media. |
| `trassd-symfony` | 5 | 2 | Symfony best practices — service config & autowiring, Doctrine, security voters, functional testing. |
| `trassd-symfony-ai` | 7 | 1 | Symfony AI stack — AI Bundle, Platform & model bridges, agents, RAG/Store, MCP, multi-agent orchestration. |

Each plugin folder under [`plugins/`](plugins/) is also self-contained (its own
`README.md`, `LICENSE`, `CHANGELOG.md`) and can be extracted to a stand-alone
repository if desired.

## License

MIT © Andreas van Hulst. See [LICENSE](LICENSE).
