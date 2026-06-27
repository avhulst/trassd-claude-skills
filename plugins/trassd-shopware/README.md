# trassd-shopware

Skills and agents that enforce **Shopware 6** plugin and app best practices:
plugin scaffolding & lifecycle, dependency injection, DAL entities &
associations, database migrations, events & subscribers, Store-API routes,
Storefront controllers, templates & JavaScript components, Administration
modules/components, scheduled tasks & messaging, Shopware Apps (manifest,
registration & webhooks, Twig app
scripts, Meteor Admin SDK), Storefront themes (theme.json, inheritance, SCSS),
the platform's Architecture Decision Records (ADRs), plus a Store quality gate.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `shopware-plugin-fundamentals` | Plugin base class, composer.json, lifecycle, config.xml |
| `shopware-dependency-injection` | services.xml/php, autowiring, tags, decoration |
| `shopware-dal-entities` | Custom entities (attribute & definition), associations, translations, reading |
| `shopware-migrations` | `MigrationStep`, update vs updateDestructive, conventions |
| `shopware-events` | Subscribers, listeners, custom events |
| `shopware-store-api-routes` | `AbstractRoute` pattern, response structs, overriding routes |
| `shopware-storefront` | Controllers, pages/pagelets, Twig customization, custom functions |
| `shopware-storefront-javascript` | Storefront JS plugin system: Plugin class, PluginManager, data-* binding, HttpClient, overrides |
| `shopware-administration` | Vue admin modules/components, overrides, base components |
| `shopware-tasks-and-messaging` | Scheduled tasks, console commands, message queue, logging |
| `shopware-app-fundamentals` | App manifest.xml, registration/lifecycle, webhooks, signature verification |
| `shopware-app-scripts` | Twig server-side app scripts: data loading, cart manipulation, custom endpoints |
| `shopware-app-admin-extensions` | Meteor Admin SDK: app admin modules, action buttons, CMS elements, snippets |
| `shopware-themes` | theme.json, configuration & inheritance, SCSS/Bootstrap variables & breakpoints, assets/icons |
| `shopware-adrs` | Binding Architecture Decision Records distilled into actionable rules (decoration, SQL-vs-DAL, events, domain exceptions, deprecations, enums, …) |

## Agents

| Agent | When to use |
|-------|-------------|
| `shopware-plugin-reviewer` | Review plugin code (DI, DAL, routes, controllers, admin) against conventions. |
| `shopware-store-quality-gate` | Audit an extension against the Shopware Store quality guidelines before submission. |
| `shopware-adr-auditor` | Audit extension code against the platform's binding ADRs (decoration, SQL-vs-DAL, domain exceptions, deprecations, enums, …). |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-shopware@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
