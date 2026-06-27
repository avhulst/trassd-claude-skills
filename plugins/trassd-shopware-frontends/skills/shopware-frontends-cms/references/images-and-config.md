# CMS images & `app.config.ts` configuration

`@shopware/cms-base-layer` bundles a Nuxt Image setup tuned for Shopware 6 and
several image-related composables. Configuration is read from `app.config.ts`
and `nuxt.config.ts`.

## NuxtImg + the Shopware provider

Replace `<img>` with `<NuxtImg>` to get a custom provider that maps Nuxt Image
modifiers to Shopware query parameters:

- `width` → `?width=`, `height` → `?height=`, `quality` → `?quality=`,
  `format` → `?format=`, `fit` → `?fit=`.

```vue
<NuxtImg
  :src="product.cover.media.url"
  preset="productDetail"
  :width="800"
  :alt="product.cover.media.alt"
/>
```

`quality`, `format` and `fit` are advanced transforms: they work out of the box
on Shopware Cloud (Fastly), but self-hosted instances need remote thumbnail
generation (e.g. FroshPlatformThumbnailProcessor). Without it, only the
predefined static thumbnails are served and these params are ignored.

### Presets (pre-configured)

`productCard`, `productDetail`, `thumbnail` (150×150), `hero`. Override or add
presets in `nuxt.config.ts`:

```ts
export default defineNuxtConfig({
  extends: ["@shopware/cms-base-layer"],
  image: {
    quality: 85,
    formats: ["avif", "webp", "jpg"],
    presets: {
      productCard: { modifiers: { format: "avif", quality: 80, fit: "cover" } },
      categoryBanner: {
        modifiers: { format: "webp", quality: 90, width: 1200, height: 400, fit: "cover" },
      },
    },
  },
});
```

> The `productCard` preset only carries URL modifiers (format/quality/fit).
> Keep `width`/`height`/`densities="1x"`/`loading="lazy"` on the component —
> presets don't reliably propagate those. Avoid `decoding`/`sizes` props on
> `NuxtImg`: they cause Vue hydration mismatches and duplicate requests.

## Responsive sizing

- `CmsElementImage` measures its container with `useElementSize()` and passes
  the size to `NuxtImg` (no fetch during SSR; one correctly-sized image after
  hydration, ×2 for retina, rounded to the nearest 100px).
- Slider elements (`CmsElementProductSlider`, `CmsElementCrossSelling`) `inject`
  the slot count via `cms-block-slot-count` to scale SSR breakpoints.

## Image placeholder

`useImagePlaceholder` generates an SVG placeholder shown while images load.

```ts
// app.config.ts — global color
export default defineAppConfig({
  imagePlaceholder: { color: "#543B95" }, // default
});
```

```vue
<script setup>
const customPlaceholder = useImagePlaceholder("#FF0000");
</script>
<template>
  <NuxtImg :placeholder="customPlaceholder" src="..." />
</template>
```

## Background image optimization

`CmsPage` (section backgrounds) and `CmsGenericBlock` (block backgrounds) read
config from `app.config.ts` and pass it to `getBackgroundImageUrl` from
`@shopware/helpers`, appending `width`/`height` (capped at 1920px),
`fit=crop,smart`, plus `format`/`quality`:

```ts
export default defineAppConfig({
  backgroundImage: {
    format: "webp", // "webp" | "avif" | "jpg" | "png"
    quality: 90,    // 0-100
  },
});
```

Setting either to `undefined`/omitting it skips that param. Like other dynamic
transforms, this needs remote thumbnail generation on self-hosted instances.

## LCP image preload

`useLcpImagePreload(sections)` scans sections in document order (section bg →
block bg → image-element media) and injects
`<link rel="preload" as="image" fetchpriority="high">` for the first image
during SSR. It is already called inside `CmsPage`. If you override `CmsPage`,
call it yourself:

```vue
<script setup lang="ts">
import { useLcpImagePreload } from "@shopware/cms-base-layer/composables/useLcpImagePreload";
import type { Schemas } from "#shopware";
const props = defineProps<{ content: Schemas["CmsPage"] }>();
useLcpImagePreload(props.content?.sections || []);
</script>
```

Preload is gated by `appConfig.lcpImagePreload` (set it to `true` to enable;
the README's changelog notes automatic preload is disabled by default in recent
versions to avoid noisy preload warnings).

## `app.config.ts` keys (summary)

- `imagePlaceholder.color` — placeholder background color.
- `backgroundImage.format` / `backgroundImage.quality` — CMS background CDN opts.
- `lcpImagePreload` — enable first-image LCP preload.
- `imageSizes` — map block slot count → responsive `sizes` value (used by
  `CmsGenericBlock` to hint child image elements).
- `unocssRuntime` — only relevant when extending
  `@shopware/unocss-design-tokens-layer`; set `false` to disable the runtime
  utility-class resolver.
