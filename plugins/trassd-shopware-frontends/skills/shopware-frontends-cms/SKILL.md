---
name: shopware-frontends-cms
description: Render Shopware CMS (Shopping Experiences) with @shopware/cms-base-layer — CmsPage, CMS sections/blocks/elements, and the useCms* composables. Triggers when rendering CMS pages or adding/overriding CMS block/element components in a Vue 3 / Nuxt 4 headless storefront.
---

# Shopware Frontends CMS (`@shopware/cms-base-layer`)

`@shopware/cms-base-layer` is a **Nuxt layer** that ships Vue components for
Shopware's *Shopping Experiences* CMS: every CMS section, block and element
rendered with utility-class markup. It is powered by `@shopware/composables`
and is fully typed — no extra packages needed.

Use it when a Store-API response (Product, Category, or Landing Page) carries a
`cmsPage` layout and you need to render it, or when you want to customize how a
specific block/element looks.

## Set up the layer

Install as a dev dependency and register it in `extends`, after the composables
layer:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  extends: ["@shopware/composables/nuxt-layer", "@shopware/cms-base-layer"],
  modules: ["@shopware/nuxt-module"],
});
```

The layer **no longer ships a default UnoCSS theme**. Choose one:

- extend `@shopware/unocss-design-tokens-layer` alongside it for the shared
  Shopware Frontends token palette + UnoCSS defaults (this layer also gives you
  a client-side UnoCSS runtime that resolves CMS utility classes set in the
  admin at runtime — enabled by default, disable with `unocssRuntime: false`
  in `app.config.ts`), **or**
- keep only `cms-base-layer` and bring your own UnoCSS / Tailwind setup.

> Some components render `RouterLink` internally — keep Vue Router installed /
> Nuxt router enabled to avoid missing-component warnings.

## Render a CMS page

All components are auto-registered by the layer, so **no imports** are needed in
templates. Render a layout by passing its `cmsPage` object to `CmsPage`:

```vue
<template>
  <CmsPage v-if="response.cmsPage" :content="response.cmsPage" />
</template>
```

`CmsPage` walks `content.sections`, renders each section, and (via
`useLcpImagePreload`) can preload the first CMS image as the LCP candidate.

## Component resolution (naming convention)

The layer maps each CMS node's `type` to a Vue component **by name** — there is
no manual registry. Knowing the rule is what lets you override or add
components.

- **Sections:** `CmsPage` resolves `CmsSection{PascalCase(section.type)}`
  (e.g. type `sidebar` → `CmsSectionSidebar`, default → `CmsSectionDefault`).
- **Blocks & elements:** `CmsGenericBlock` / `CmsGenericElement` delegate to the
  `resolveCmsComponent` helper from `@shopware/composables`. It builds the name
  `PascalCase("Cms-{kind}-{type}")` where `{kind}` is chosen from the node's
  `apiAlias` (`cms_block` → `Block`, `cms_section` → `Section`, otherwise
  `Element`). So block type `image-text` → `CmsBlockImageText`; element type
  `product-listing` → `CmsElementProductListing`.

If no matching component is found, the layer renders `CmsNoComponent` and (in
dev) logs which `*.vue` name to create. See
[references/resolution-and-overrides.md](references/resolution-and-overrides.md)
for the full list of built-in block/element names and details.

## Override or add a block/element component

Because resolution is name-based, **shadowing wins**: create a `.vue` file with
the exact resolved name in your project's `~/components` directory (or wherever
your Nuxt config auto-imports from). Nuxt prefers your component over the
layer's.

- Override an existing block: add `components/CmsBlockImageText.vue`.
- Override an element: add `components/CmsElementImage.vue`.
- Add support for a custom CMS type: implement the component under the resolved
  name (e.g. a custom element type `my-widget` → `CmsElementMyWidget.vue`).

Every public block/element receives a `content` prop typed against the
Schemas — e.g. a block gets `Schemas["CmsBlock"]`, an element gets
`Schemas["CmsSlot"]`. Read config and slots through the `useCms*` composables
below rather than poking at the raw object.

> **Internal `Sw*` components** (e.g. `SwSlider`, `SwProductCard`,
> `SwSharedPrice`) are shared building blocks, **not public API**. You can
> shadow them the same way (e.g. one `SwSharedPrice.vue` changes price display
> everywhere), but you then own tracking upstream changes.

## The `useCms*` composables

Re-exported from `@shopware/composables` (the layer's components use them
internally; use them in your own/overridden components):

- **`useCmsSection(section)`** — exposes the section and its blocks; use it to
  iterate blocks in a custom section component.
- **`useCmsBlock(block)`** — exposes the block and `getSlotContent(slotName)` to
  fetch a named slot (e.g. `getSlotContent("main")`) when rendering a custom
  block.
- **`useCmsElementConfig(element)`** — typed accessor for an element's
  `config.*` values set in the admin.
- **`useCmsElementImage(element)`** — image element helpers (`containerStyle`,
  `imageContainerAttrs`, plus `isVideoElement` / `mimeType` checks).
- **`useCmsMeta`** / **`useCmsTranslations`** — page meta and translation
  helpers.

Short example of a custom block reading a slot:

```vue
<script setup lang="ts">
import type { Schemas } from "#shopware";
const props = defineProps<{ content: Schemas["CmsBlock"] }>();
const { getSlotContent } = useCmsBlock(props.content);
const left = getSlotContent("left");
</script>

<template>
  <div class="grid grid-cols-2">
    <CmsGenericElement :content="left" />
  </div>
</template>
```

`CmsGenericBlock` / `CmsGenericElement` are the building blocks for recursing
into child slots from your own components.

## Images

Replace `<img>` with `<NuxtImg>` to get the layer's Shopware Image provider
(maps `width`/`height`/`quality`/`format`/`fit` to Shopware query params) and
preset support (`productCard`, `productDetail`, `thumbnail`, `hero`). Section
and block background images are auto-optimized via `getBackgroundImageUrl`. See
[references/images-and-config.md](references/images-and-config.md) for presets,
placeholders, LCP preload, and `app.config.ts` keys.
