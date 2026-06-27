# Kit CSS / asset setup

Each official kit requires TailwindCSS plus a small per-kit setup in
`assets/styles/app.css`. The AssetMapper and Webpack Encore variants differ only
in the import paths (AssetMapper imports from `../vendor/...`; Encore imports the
npm package name). Always re-check the kit's own `INSTALL.md` for the current,
full snippet — the Toolkit is experimental and these can change.

## Shadcn UI

1. Install the CSS packages:

   ```bash
   # AssetMapper
   php bin/console importmap:require shadcn/dist/tailwind.css tw-animate-css/dist/tw-animate.css

   # Webpack Encore
   npm install shadcn tw-animate-css
   ```

2. In `assets/styles/app.css`, import Tailwind, then the kit CSS, and add the
   kit's `@theme inline` tokens and `:root` / `.dark` CSS variables:

   ```css
   @import 'tailwindcss';

   /* With AssetMapper... */
   @import '../vendor/tw-animate-css/dist/tw-animate.css';
   @import '../vendor/shadcn/dist/tailwind.css';
   /* ... or with Webpack Encore */
   /* @import "tw-animate-css"; */
   /* @import "shadcn/tailwind.css"; */

   @custom-variant dark (&:is(.dark *));

   /* @theme inline { ... }  — color + radius tokens
      :root { ... } / .dark { ... } — light/dark palette
      @layer base { ... } — border/background defaults
      (full token list is in the kit's INSTALL.md) */
   ```

The Shadcn kit defines a large set of design tokens (background, foreground,
card, popover, primary, secondary, muted, accent, destructive, border, input,
ring, chart-*, sidebar-*, and radius scale). Copy the complete block from the
kit's `INSTALL.md` when setting up.

## Flowbite (v4)

Requires TailwindCSS **and** Flowbite v4.

1. Install Flowbite:

   ```bash
   # AssetMapper
   php bin/console importmap:require flowbite

   # Webpack Encore
   npm install flowbite
   ```

2. In `assets/styles/app.css`, import Tailwind and point Tailwind at the Flowbite
   source, then add the kit's `@theme` and `.dark` variables:

   ```css
   @import 'tailwindcss';

   /* With AssetMapper... */
   @source "../vendor/flowbite";
   /* ... or with Webpack Encore */
   /* @source "../../node_modules/flowbite"; */

   @custom-variant dark (&:is(.dark *));

   /* @theme { ... }  — fonts, text/leading/tracking, radius, border-width,
      and an extensive color system (body/heading, brand, status, neutral,
      border colors)
      .dark { ... } — dark-mode overrides of those variables
      (full token list is in the kit's INSTALL.md) */
   ```

3. Load Flowbite's JS by adding to `assets/app.js`:

   ```js
   import 'flowbite';
   ```
