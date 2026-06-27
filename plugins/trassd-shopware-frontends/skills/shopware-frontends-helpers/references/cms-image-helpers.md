# CMS & image-URL helpers — usage

Detailed usage for the image/URL and CMS-class helpers from `@shopware/helpers`.
All examples use real exported names; signatures are illustrative.

## `getBackgroundImageUrl`

Generates an optimized CSS `url()` value for a CMS **background** image. It
extracts the raw URL, derives dimensions from the media metadata
(`backgroundMedia.metaData.width/height`), and appends image transformation
query parameters. Used internally by CMS page/block rendering.

```ts
import { getBackgroundImageUrl } from "@shopware/helpers";

const optimizedUrl = getBackgroundImageUrl(
  "url(https://cdn.shopware.store/.../image.jpg)",
  cmsBlockOrSection, // object with backgroundMedia.metaData.width/height
  { format: "webp", quality: 85 }, // optional BackgroundImageOptions
);
// => 'url("https://cdn.shopware.store/.../image.jpg?width=1000&fit=crop,smart&format=webp&quality=85")'
```

Parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `url` | `string` | CSS `url()` string containing the background image URL |
| `element` | `{ backgroundMedia?: { metaData?: { width?: number; height?: number } } }` | CMS section or block with media metadata |
| `options` | `BackgroundImageOptions` (optional) | Format and quality settings |

`BackgroundImageOptions`:

```ts
type BackgroundImageOptions = {
  format?: string; // e.g. "webp" | "avif" | "jpg" | "png"
  quality?: number; // 0-100
};
```

When `format` or `quality` are provided they are appended as `&format=` /
`&quality=` query parameters; if omitted, only dimension and `fit` parameters
are applied.

## `generateCdnSrcSet`

Builds an HTML `srcset` string using CDN width-based resizing — a good fallback
when media has no pre-generated thumbnails.

```ts
import { generateCdnSrcSet } from "@shopware/helpers";

const srcset = generateCdnSrcSet(
  "https://cdn.shopware.store/.../image.jpg",
  [400, 800, 1200, 1600], // optional, these are the defaults
  { format: "webp", quality: 85 }, // optional
);
// => "https://cdn.shopware.store/.../image.jpg?width=400&fit=crop,smart&format=webp&quality=85 400w, ...1600w"
```

Returns `undefined` if the source is falsy or URL parsing fails.

## `buildCdnImageUrl`

Builds a single optimized CDN image URL from rendered element dimensions. Adds
`width` or `height` (whichever is larger) rounded up to the nearest 100px, plus
`fit=crop,smart`.

```ts
import { buildCdnImageUrl } from "@shopware/helpers";

const url = buildCdnImageUrl(
  "https://cdn.shopware.store/.../image.jpg",
  { width: 724, height: 760 },
  { format: "webp", quality: 85 }, // optional
);
// => "https://cdn.shopware.store/.../image.jpg?height=800&fit=crop,smart"
```

Returns an empty string if the source is falsy; returns the original source if
URL parsing fails.

## `helpersCssClasses` — CMS reusable classes

`helpersCssClasses` is an array of CMS class names used across packages (defined
in the CMS layout-classes helper). Use it as a `safelist` so dynamically applied
visibility classes are not purged by your atomic-CSS engine.

```ts
import { helpersCssClasses, type HelpersCssClasses } from "@shopware/helpers";

// UnoCSS config
export default defineConfig({
  safelist: helpersCssClasses,
});

// Type-safe visibility mapping example
const visibilityMap: Record<"mobile" | "tablet" | "desktop", HelpersCssClasses> = {
  mobile: "max-md:hidden",
  tablet: "md:max-lg:hidden",
  desktop: "lg:hidden",
};
```
