# trassd-shopware-frontends

Skills and agents for **Shopware Frontends** — the Vue 3 / Nuxt 4 / TypeScript
framework for headless Shopware 6 storefronts. Covers project setup & templates,
the API client (Store + Admin), Vue composables, the CMS base layer, API type
generation, and the framework-agnostic helpers.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `shopware-frontends-project-setup` | Templates, `@shopware/nuxt-module` config, env, type-gen, Nuxt layers/extending |
| `shopware-frontends-api-client` | `createAPIClient` / `createAdminAPIClient`, typed `invoke`, context token, error handling |
| `shopware-frontends-composables` | `useProduct/useCart/useUser/useCheckout/useListing/useNavigation` + shopware context |
| `shopware-frontends-cms` | `cms-base-layer`: `CmsPage`, block/element resolution, overrides, `useCms*` |
| `shopware-frontends-api-gen` | `@shopware/api-gen` type generation workflow |
| `shopware-frontends-helpers` | Price formatting, image/URL helpers (`getBackgroundImageUrl`), translation |

## Agents

| Agent | When to use |
|-------|-------------|
| `shopware-frontends-code-reviewer` | Review composables/API-client/Nuxt/CMS usage against framework conventions. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-shopware-frontends@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
