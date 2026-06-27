# Extending a base template with Nuxt layers

The pattern behind `vue-starter-template-extended`: build a brand-specific storefront on
top of a base template using Nuxt's layer mechanism, keeping only the customizations.
Distilled from the Shopware Frontends AGENTS guide and the extended starter template.

## The layer pattern

Use `extends` in `nuxt.config.ts` to inherit everything from a base layer. The source can
be a local path or an npm package.

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  extends: ["../vue-starter-template"], // or an npm package
  // ...your customizations
});
```

A child layer typically declares the base template as a (workspace) dependency and
includes `@shopware/cms-base-layer` for CMS components.

Core Shopware packages are themselves layers and are commonly composed the same way in the
base template, e.g.:

```ts
extends: [
  "@shopware/composables/nuxt-layer",
  "@shopware/cms-base-layer",
],
```

## What you inherit

Everything from the base layer is automatically available — you do not redeclare it:

- Pages (e.g. `FrontendNavigationPage`, `FrontendDetailPage`)
- Layouts (headers, footers, navigation)
- Forms (login, checkout, account)
- Shared components (modals, notifications)
- Composables

## Overriding components

To replace a base component, create a component with the **same name** under your
project's `app/components/` directory. Yours takes precedence over the base layer's:

```
your-project/
  app/
    components/
      SwProductCard.vue   # overrides the base SwProductCard
```

## Customizing layer config and theme

Use `app/app.config.ts` with `defineAppConfig` to adjust layer settings such as the image
placeholder color:

```ts
// app/app.config.ts
export default defineAppConfig({
  imagePlaceholder: {
    color: "#B38A65", // brand color
  },
});
```

Brand styling (UnoCSS tokens, etc.) is configured through the template's `uno.config.ts`.
The Shopware connection (`endpoint` / `accessToken`) is set in the child's
`runtimeConfig.public.shopware` just like a standalone project.

## Why use layers

- Inherit all features from the base template with minimal code duplication.
- Override only the specific components or settings that differ.
- Pick up improvements automatically when the base template updates.
- Maintain multiple store variants from a single shared base, and test customizations
  without modifying the base.

## When not to use it

If you need deep structural changes rather than targeted overrides, or you aren't using
Nuxt at all (e.g. the `vue-vite-blank` / Astro routes), the layer approach doesn't apply —
start from the appropriate standalone template instead.
