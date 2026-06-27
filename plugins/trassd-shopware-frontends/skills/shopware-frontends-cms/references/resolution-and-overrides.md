# CMS component resolution & overrides

How `@shopware/cms-base-layer` turns a Shopware CMS layout into Vue components,
and how to shadow or extend it. All names below are derived by the layer's own
resolution logic — match them exactly.

## The resolution rule

Given a CMS node, the resolved component name is **PascalCase of a prefixed
type string**:

- **Section** (`CmsPage` resolves directly):
  `CmsSection{PascalCase(section.type)}`
  - `default` → `CmsSectionDefault`
  - `sidebar` → `CmsSectionSidebar`
- **Block / Element** (`CmsGenericBlock` / `CmsGenericElement` use
  `resolveCmsComponent` from `@shopware/composables`):
  `PascalCase("Cms-{kind}-{type}")`, where `{kind}` comes from the node's
  `apiAlias`:
  - `apiAlias === "cms_block"` → `Block`
  - `apiAlias === "cms_section"` → `Section`
  - anything else → `Element`

Examples:

| CMS node | `type` | resolved component |
|----------|--------|--------------------|
| block | `image-text` | `CmsBlockImageText` |
| block | `product-listing` | `CmsBlockProductListing` |
| element | `image` | `CmsElementImage` |
| element | `product-slider` | `CmsElementProductSlider` |
| section | `sidebar` | `CmsSectionSidebar` |

When nothing resolves, the layer renders `CmsNoComponent` and, in dev mode,
warns with the exact `*.vue` file name you should create.

## Built-in components (shadow by name)

Sections: `CmsSectionDefault`, `CmsSectionSidebar`.

Blocks (selection — match the file name): `CmsBlockText`, `CmsBlockImage`,
`CmsBlockImageText`, `CmsBlockImageSlider`, `CmsBlockImageGallery`,
`CmsBlockProductListing`, `CmsBlockProductSlider`, `CmsBlockProductHeading`,
`CmsBlockCrossSelling`, `CmsBlockCategoryNavigation`, `CmsBlockForm`,
`CmsBlockCustomForm`, `CmsBlockHtml`, `CmsBlockYoutubeVideo`,
`CmsBlockVimeoVideo`, `CmsBlockTextHero`, `CmsBlockTextOnImage`,
`CmsBlockSidebarFilter`, `CmsBlockGalleryBuybox`, and the various
`CmsBlockImage*` / `CmsBlockText*` layout variants. `CmsBlockDefault` is the
generic fallback layout.

Elements: `CmsElementText`, `CmsElementImage`, `CmsElementImageGallery`,
`CmsElementImageSlider`, `CmsElementProductBox`, `CmsElementProductListing`,
`CmsElementProductSlider`, `CmsElementProductName`,
`CmsElementProductDescriptionReviews`, `CmsElementBuyBox`,
`CmsElementCrossSelling`, `CmsElementCategoryNavigation`,
`CmsElementManufacturerLogo`, `CmsElementSidebarFilter`, `CmsElementForm`,
`CmsElementCustomForm`, `CmsElementHtml`, `CmsElementYoutubeVideo`,
`CmsElementVimeoVideo`.

> The authoritative, version-specific list lives in the upstream package and
> the framework docs ("Available components"). Verify the exact name in your
> installed version with Vue Devtools, or trust the dev-mode warning which
> prints the precise file name to create.

## Overriding a built-in component

Resolution is purely name-based, so a same-named component in your app shadows
the layer's. Place the file where your Nuxt config auto-imports components
(default `~/components`):

```vue
<!-- ~/components/CmsBlockImageText.vue -->
<script setup lang="ts">
import type { Schemas } from "#shopware";
const props = defineProps<{ content: Schemas["CmsBlock"] }>();
const { getSlotContent } = useCmsBlock(props.content);
const text = getSlotContent("right");
const image = getSlotContent("left");
</script>

<template>
  <section class="my-image-text">
    <CmsGenericElement :content="image" />
    <CmsGenericElement :content="text" />
  </section>
</template>
```

```vue
<!-- ~/components/CmsElementImage.vue -->
<script setup lang="ts">
import type { Schemas } from "#shopware";
const props = defineProps<{ content: Schemas["CmsSlot"] }>();
const { containerStyle, imageContainerAttrs, isVideoElement, mimeType } =
  useCmsElementImage(props.content);
</script>

<template>
  <figure :style="containerStyle">
    <NuxtImg v-bind="imageContainerAttrs" />
  </figure>
</template>
```

`useCmsElementImage` returns (typed via `UseCmsElementImage` /
`ImageContainerAttrs`): `containerStyle`, `imageContainerAttrs`,
`isVideoElement`, `mimeType` — verify against the types in your installed
version.

## Adding support for a custom CMS type

If your Shopware instance ships a custom block or element type, the layer will
fall back to `CmsNoComponent`. Implement a component under the resolved name to
render it:

- custom element type `my-widget` → `~/components/CmsElementMyWidget.vue`
- custom block type `promo-banner` → `~/components/CmsBlockPromoBanner.vue`

Inside, recurse into child slots with `CmsGenericElement` and read config with
`useCmsElementConfig(content)`.

## Internal `Sw*` components — not public API

Components prefixed `Sw` (e.g. `SwSlider`, `SwProductCard`, `SwSharedPrice`,
`SwProductPrice`, `SwPagination`) are shared internals used across public
blocks/elements. They can be shadowed the same way — for instance a single
`~/components/SwSharedPrice.vue` changes price rendering everywhere it is
auto-imported. **Once you override an internal component you must track its
upstream changes yourself**, since it is outside the stable public API.
