# trassd-contao

Skills and agents that enforce **Contao CMS** extension/bundle best practices:
bundle structure, DCA configuration, fragment controllers (content elements &
modules), page controllers, Twig templates, insert tags, migrations,
DataContainer security voters, hooks, models, image processing, and translations.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `contao-bundle-structure` | Bundle class, composer (contao-bundle), Manager plugin, namespaces, publishing |
| `contao-dca-config` | DCA tables/palettes/fields/callbacks, PaletteManipulator |
| `contao-fragment-controllers` | Content elements, frontend & back end modules (`#[AsContentElement]`/`#[AsFrontendModule]`) |
| `contao-page-controllers` | Custom page types (`#[AsPage]`), back end routes |
| `contao-twig-templates` | Template hierarchy, overriding, modern vs legacy templates |
| `contao-insert-tags` | `#[AsInsertTag]`, flags, safe output |
| `contao-migrations` | `MigrationInterface`/`AbstractMigration`, `contao.migration` tag, `contao:migrate` |
| `contao-security-voters` | DataContainer voters, `ContaoCorePermissions`, isGranted |
| `contao-hooks` | `#[AsHook]`, listeners, hook signatures |
| `contao-models` | Models (Active Record over DCA), collections, customization, enums |
| `contao-image-processing` | Image/picture factory, Image Studio, image sizes |
| `contao-translations` | XLIFF/PHP language files, contao/languages, translator |
| `contao-testing` | PHPUnit: ContaoTestCase (unit), FunctionalTestCase (fixtures, DB reset), test suites |
| `contao-quality-tooling` | QA gate: ECS (contao/easy-coding-standard), PHPStan, Rector, depcheck + composer scripts |
| `contao-caching` | HTTP/shared caching, response tagging & invalidation, fragment caching |
| `contao-cron-jobs-messaging` | Cron jobs (`#[AsCronJob]`), the Jobs API, async Messenger handlers |

## Agents

| Agent | When to use |
|-------|-------------|
| `contao-extension-reviewer` | Review bundle/services/controllers/models/hooks against Contao conventions & coding standards. |
| `contao-dca-linter` | Audit DCA palette/field/callback correctness. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-contao@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
