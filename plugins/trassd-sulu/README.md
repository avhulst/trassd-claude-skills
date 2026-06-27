# trassd-sulu

Skills and agents that enforce **Sulu CMS** best practices: page & snippet
templates (XML structure + property types + Twig), content architecture & smart
content, webspace configuration & localization, extending the admin UI,
extending Doctrine entities & dimension content, website rendering (controllers,
Twig, HTTP cache), security & permissions, and media handling.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `sulu-page-templates` | Page/snippet template XML structure + property types paired with their Twig files |
| `sulu-content-architecture` | Structures, properties, content types, smart content & data providers |
| `sulu-webspaces` | Webspace XML (portals, localizations, URLs, segments) and locale/localization providers |
| `sulu-extend-admin` | Admin classes, navigation items, view builders, form/list metadata, admin frontend build |
| `sulu-extend-entities` | Persistence component, entity extensions, dimension content, extensible entities |
| `sulu-website-rendering` | Custom controllers, Sulu Twig attributes/extensions, request analyzer, HTTP caching |
| `sulu-security-permissions` | Roles, permissions, security contexts, password policy |
| `sulu-media` | Image formats, media configuration, external storage adapters, properties providers |

## Agents

| Agent | When to use |
|-------|-------------|
| `sulu-code-reviewer` | Review Sulu code (templates, content types, admin classes, controllers, entities, webspace config) against Sulu conventions. |
| `sulu-template-linter` | Audit page/snippet template XML and their Twig counterparts for correctness. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-sulu@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
