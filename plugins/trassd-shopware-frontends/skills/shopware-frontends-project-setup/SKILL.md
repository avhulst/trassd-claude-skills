---
name: shopware-frontends-project-setup
description: >-
  Set up a Shopware Frontends storefront — choosing a starter template, installing and
  configuring @shopware/nuxt-module (endpoint, accessToken), wiring environment variables,
  generating API types with @shopware/api-gen, and extending a base template via Nuxt layers.
  Triggers when scaffolding a new Shopware Frontends / headless Shopware 6 storefront, editing
  the `shopware` options in nuxt.config.ts, setting up .env / type generation, or building a
  brand variant on top of an existing template.
---

# Shopware Frontends — Project Setup

Shopware Frontends is a Vue 3 / Nuxt 4 / TypeScript framework for building headless
Shopware 6 storefronts. It ships as a pnpm + Turbo monorepo with core packages
(`@shopware/api-client`, `@shopware/composables`, `@shopware/helpers`,
`@shopware/cms-base-layer`, `@shopware/nuxt-module`, `@shopware/api-gen`) and a set of
starter templates. This skill covers picking a starting point and getting a storefront
connected to a Shopware 6 instance with type-safe API access.

## 1. Pick a template

Start from a template instead of an empty project. Choose by need:

| Need | Template |
|------|----------|
| Learn all features / full reference (UnoCSS, i18n, CMS) | `vue-demo-store` |
| Start a production project from a clean base | `vue-starter-template` |
| Extend an existing template (brand variant) via Nuxt layers | `vue-starter-template-extended` |
| Minimal Nuxt setup, core packages only, no styling/i18n | `vue-blank` |
| No SSR / no Nuxt (plain Vite + Vue SPA) | `vue-vite-blank` |
| Astro framework integration | `astro` |

Rules of thumb:
- For most real storefronts, begin with **`vue-starter-template`** — it has the core
  packages, UnoCSS, i18n, and type-generation already wired but no demo content.
- Use **`vue-demo-store`** only as a feature reference; it carries demo content you'll
  have to strip.
- Reach for **`vue-vite-blank`** (or a custom Vue project) when you don't want Nuxt/SSR.
  Non-Nuxt projects can't use `@shopware/nuxt-module` and must wire the API client and
  composables manually.
- Pick **`vue-starter-template-extended`** when you want a brand-specific storefront on
  top of an existing base — see section 5.

Environment: Node.js 20.x or 22.x and pnpm 10.x (use Corepack). Install with `pnpm i`;
when working inside the monorepo, build packages first with
`pnpm run build --filter='./packages/*'`.

## 2. Install and register @shopware/nuxt-module

For any Nuxt-based template the storefront talks to Shopware through
`@shopware/nuxt-module`, which bundles and pre-configures the API client and composables
(composables are auto-imported, so no manual `import` is needed).

```bash
pnpm add -D @shopware/nuxt-module
```

Register it in the `modules` array of `nuxt.config.ts`:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ["@shopware/nuxt-module"],
});
```

## 3. Configure the Shopware connection

The module needs two values: the **Store API `endpoint`** and a **Sales Channel
`accessToken`** (Shopware admin → Settings → Sales Channel → API access). There are two
equivalent places to set them; `runtimeConfig` always overrides the top-level `shopware`
key.

Simple form (top-level `shopware` key):

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ["@shopware/nuxt-module"],
  shopware: {
    endpoint: "https://your-shop.com/store-api/",
    accessToken: "YOUR_SALES_CHANNEL_ACCESS_TOKEN",
  },
});
```

Recommended form (via `runtimeConfig.public`, what the starter templates use) — this is
what lets `.env` values override the config at runtime, and exposes the values to the
client:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ["@shopware/nuxt-module"],
  runtimeConfig: {
    public: {
      shopware: {
        endpoint: "https://your-shop.com/store-api/",
        accessToken: "YOUR_SALES_CHANNEL_ACCESS_TOKEN",
        // optional: storefront domain used for local customer registration
        devStorefrontUrl: "",
      },
    },
  },
});
```

Notes:
- Values under `runtimeConfig.public` are available on both server and client; plain
  `runtimeConfig` keys are server-only.
- `devStorefrontUrl` is only needed for local development where `localhost` doesn't match
  a configured Sales Channel domain; leave it empty in production (the storefront falls
  back to `window.location.origin`).
- Custom default API headers can be set under `runtimeConfig.apiClientConfig.headers`
  (SSR) and `runtimeConfig.public.apiClientConfig.headers` (client). See
  [references/nuxt-module-config.md](references/nuxt-module-config.md).

## 4. Environment variables and type generation

### .env

Copy `.env.template` to `.env` and fill in your instance. The starter template maps
runtime config automatically through Nuxt's `NUXT_PUBLIC_*` convention, so these env
vars override the matching `runtimeConfig.public.shopware.*` keys without extra wiring:

```bash
# .env (copied from .env.template)
NUXT_PUBLIC_SHOPWARE_ENDPOINT="https://your-shop.com/store-api/"
NUXT_PUBLIC_SHOPWARE_ACCESS_TOKEN="YOUR_SALES_CHANNEL_ACCESS_TOKEN"
NUXT_PUBLIC_SHOPWARE_DEV_STOREFRONT_URL=""

# used only by type generation (@shopware/api-gen), not at runtime
OPENAPI_JSON_URL="https://your-shop.com"        # instance base, no /store-api/ suffix
OPENAPI_ACCESS_KEY="YOUR_SALES_CHANNEL_ACCESS_KEY"
```

### Generate API types

Generate types tailored to your Shopware instance so the API client and composables are
fully typed against your actual schema (including commercial/custom fields):

```bash
pnpm run generate-types   # runs: shopware-api-gen generate --apiType=store
```

This uses `@shopware/api-gen`, driven by `OPENAPI_JSON_URL` / `OPENAPI_ACCESS_KEY` and the
`api-gen.config.json` file (which can list schema override patches). Generated types land
in the project's `api-types/` folder.

### Register custom types with #shopware

To make the generated types flow into `apiClient` everywhere, register them on the
`#shopware` module via a `shopware.d.ts` file at the project root. See
[references/type-generation.md](references/type-generation.md).

## 5. Extend a base template with Nuxt layers

To build a brand variant without duplicating code, extend an existing template using
Nuxt's `extends` (the approach of `vue-starter-template-extended`). The child layer keeps
only customizations and overrides.

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  extends: ["../vue-starter-template"], // local path or an npm package
  runtimeConfig: {
    public: {
      shopware: {
        endpoint: "https://your-shop.com/store-api/",
        accessToken: "YOUR_SALES_CHANNEL_ACCESS_TOKEN",
      },
    },
  },
});
```

What you get and how to customize:
- **Inheritance** — all components, pages, layouts, forms, and composables from the base
  layer are automatically available; you don't re-declare them.
- **Component overriding** — create a component with the same name under `app/components/`
  in your project to override the base one (e.g. `app/components/SwProductCard.vue`
  overrides the base `SwProductCard`).
- **Theme/config** — adjust layer settings (e.g. the image placeholder color) in
  `app/app.config.ts` with `defineAppConfig`. Style brand tokens via the template's
  `uno.config.ts`.

```ts
// app/app.config.ts
export default defineAppConfig({
  imagePlaceholder: {
    color: "#B38A65", // brand color
  },
});
```

Note: core packages also ship as layers and are commonly composed via `extends` in the
starter template — e.g. `@shopware/composables/nuxt-layer` and `@shopware/cms-base-layer`.

See [references/layers-and-overrides.md](references/layers-and-overrides.md) for the full
layer pattern, override mechanics, and the trade-offs.

## Quick checklist for a new storefront

1. Scaffold from the right template (usually `vue-starter-template`); `pnpm i`.
2. Ensure `@shopware/nuxt-module` is in `modules`.
3. Set `endpoint` + `accessToken` (prefer `runtimeConfig.public.shopware`).
4. Copy `.env.template` → `.env`; fill `NUXT_PUBLIC_SHOPWARE_*` and `OPENAPI_*`.
5. Run `pnpm run generate-types`; register them via `shopware.d.ts` / `#shopware`.
6. Run the dev server and verify a composable (e.g. `useSessionContext`) connects.
7. For brand variants, extend the base via `extends` and override only what differs.
