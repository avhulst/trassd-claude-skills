---
name: shopware-frontends-helpers
description: >-
  Use @shopware/helpers — framework-agnostic utilities for price formatting,
  URL/CMS image handling (e.g. getBackgroundImageUrl), translation, and data
  transformation in headless Shopware 6 storefronts. Triggers when formatting
  prices, building CMS/media image URLs, resolving translated entity
  properties, or transforming Shopware API data in the storefront — instead of
  re-implementing that logic by hand.
---

# Shopware Frontends — `@shopware/helpers`

`@shopware/helpers` is the framework-agnostic utility layer of Shopware
Frontends. It ships **pure functions** (no Vue/Nuxt dependency, easily testable)
that transform Shopware 6 Store-API data into display-ready values. It builds
after `api-client` and before `composables` in the package graph.

## Rule of thumb

**Reach for a helper before re-implementing.** Price formatting, image-URL
optimization, thumbnail/srcset generation, translation fallback, and product/
category/CMS data extraction are already solved here as pure functions. Import
the named export from `@shopware/helpers`; do not hand-roll equivalents in
components or composables.

```ts
import { getFormattedPrice, getProductName } from "@shopware/helpers";
```

## What's available (grouped)

### Price formatting
- `getFormattedPrice` — format a numeric price for display.
- `getProductRealPrice`, `getProductCalculatedListingPrice`,
  `getProductFromPrice`, `getProductTierPrices` (type `TierPrice`) — derive the
  correct price figures from a product.
- `getProductFreeShipping`, `isProductOnSale` — pricing-related predicates.

### URL & image handling
- `getBackgroundImageUrl` — optimized CSS `url()` for **CMS background images**
  (takes `BackgroundImageOptions { format?, quality? }`). See
  [references/cms-image-helpers.md](references/cms-image-helpers.md).
- `buildCdnImageUrl`, `generateCdnSrcSet` — on-the-fly CDN resizing for a single
  URL or a responsive `srcset`. See
  [references/cms-image-helpers.md](references/cms-image-helpers.md).
- Media helpers: `getMedia`, `getBiggestThumbnailUrl`,
  `getSmallestThumbnailUrl`, `getSrcSetForMedia`, `downloadFile`.
- Entity URLs/routes: `getProductUrl`, `getProductRoute`, `getCategoryUrl`,
  `getCategoryRoute`, `getMainImageUrl`, `getCategoryImageUrl`.
- Path utilities: `relativeUrlSlash`, `urlIsAbsolute`, `getRouteFromPathInfo`,
  `buildUrlPrefix`, `encodeUrlPath`, `normalizePath`, `isTechnicalPath`.

### Translation
- `getTranslatedProperty` — read a translatable entity field with fallback;
  prefer this over reaching into raw `translated.*` objects.
- `getCmsTranslate` — resolve CMS snippet/label translations.
- `getLanguageName` — display name for a language.

### CMS & data transformation
- `getCmsEntityObject`, `isProduct`, `isCategory`, `isLandingPage` — narrow a
  CMS page response to its main entity.
- `getCmsLayoutConfiguration` (type `LayoutConfiguration`), `getCmsBreadcrumbs`,
  `getCategoryBreadcrumbs`, `getProductListingFromCmsPage`, `isMaintenanceMode`.
- `helpersCssClasses` (type `HelpersCssClasses`) — shared CMS visibility/utility
  class names; use as a UnoCSS/Tailwind `safelist` so dynamically applied
  classes survive purging. See
  [references/cms-image-helpers.md](references/cms-image-helpers.md).
- Listing/product extraction: `getListingFilters`, `getProductName`,
  `getProductManufacturerName`, `getProductReviews`, `getProductRatingAverage`,
  `isProductTopSeller`.

## Conventions

- These are **pure, framework-agnostic functions** — call them from components,
  composables, or plain TS alike; keep them side-effect free downstream.
- Public exports are documented with JSDoc; respect them as a stable API and do
  not break signatures without a major version bump.
- They are unit-test friendly — when adding storefront logic, prefer pushing
  pure transformation into a helper-style function over inlining it in a
  component.
