---
name: shopware-themes
description: Build a Shopware 6 theme — the theme.json manifest, theme configuration & inheritance, SCSS styling (overriding Bootstrap variables and breakpoints), and bundling assets/icons. Triggers when creating a theme, editing theme.json, or customizing storefront SCSS/assets in a theme.
---

# Shopware 6 Themes

A theme customizes the **Storefront** appearance only — no backend/PHP logic. It is
technically a plugin (or, on Cloud, an app) that **implements
`Shopware\Storefront\Framework\ThemeInterface`**. Implementing that interface is what
makes it appear in the Theme Manager and become assignable **per sales channel**
(plugins/apps are global). The Storefront is a skin on top of **Bootstrap 5** — anything
Bootstrap can do, the Storefront can do.

Use a theme to override Twig templates, add SCSS/CSS and JS, define configurable
settings in the Administration, and control inheritance order. If your extension needs
PHP logic, use a plugin instead.

## Creating a theme

Scaffold, register, install, and assign:

```bash
bin/console theme:create SwagBasicExampleTheme      # scaffold plugin + theme.json
bin/console plugin:refresh                          # make Shopware aware of it
bin/console plugin:install --activate SwagBasicExampleTheme
bin/console theme:change                            # assign to a sales channel (interactive)
```

Name the plugin in **UpperCamelCase** with a company prefix (e.g. `SwagBasicExampleTheme`).
The generated layout:

```
src/
├── Resources/
│   ├── app/storefront/
│   │   ├── dist/storefront/js/...    # compiled JS (ship this)
│   │   └── src/
│   │       ├── assets/
│   │       ├── main.js               # JS entry
│   │       └── scss/{base.scss, overrides.scss}
│   └── theme.json                    # manifest
└── SwagBasicExampleTheme.php         # bundle implementing ThemeInterface
```

## theme.json manifest

Lives at `<plugin root>/src/Resources/theme.json`. Core keys:

```javascript
{
  "name": "SwagBasicExampleTheme",
  "author": "Shopware AG",
  "description": { "en-GB": "My custom theme", "de-DE": "Mein Theme" },
  "previewMedia": "app/storefront/dist/assets/defaultThemePreview.jpg",
  "views":  ["@Storefront", "@Plugins", "@SwagBasicExampleTheme"],
  "style":  ["app/storefront/src/scss/overrides.scss", "@Storefront", "app/storefront/src/scss/base.scss"],
  "script": ["@Storefront", "app/storefront/dist/storefront/js/swag-basic-example-theme/swag-basic-example-theme.js"],
  "asset":  ["@Storefront", "app/storefront/src/assets"],
  "config": { "fields": { "sw-color-brand-primary": { "value": "#00ff00" } } },
  "configInheritance": ["@Storefront", "@OtherTheme"]
}
```

- **`views`** — Twig template inheritance order.
- **`style`** — SCSS compilation order. Reference whole namespaces (`@Storefront`,
  `@OtherTheme`) or single files (`@OtherTheme/app/storefront/src/scss/custom.scss`).
  Order matters: put `overrides.scss` (variable overrides) **before** `@Storefront`.
- **`script`** — JS load order; reference the **compiled** files under `dist/`.
- **`asset`** — paths to images/fonts; add `@Storefront` to reuse default assets.
- **`config`** — Administration-editable settings (see below).
- **`configInheritance`** — extra themes whose config fields/snippets are inherited
  (available since 6.4.8.0). `@Storefront` is always inherited even if omitted.

After any edit to `theme.json`, run `bin/console theme:refresh` to apply structural
changes (it also rebuilds inheritance relationships).

## Config fields

`config.fields.<technicalName>` defines a setting. The key is also the SCSS variable
name used to access it. Each field accepts: `type` (`color`, `text`, `number`,
`fontFamily`, `media`, `checkbox`, `switch`, `url`), `value` (default), `editable`,
`tab` / `block` / `section` (grouping in the Theme Manager), `custom` (passthrough data,
e.g. a select component + options), `scss: false` (don't inject as SCSS var), and
`fullWidth`. As of v6.8 field labels/helpText are translated via Administration snippet
keys (`sw-theme.<technicalName>.<tab>.<block>.<section>.<field>.label`) rather than inline
`label` arrays.

You always inherit the Storefront config, so only specify the values you change.
See [references/theme-json.md](references/theme-json.md) for full field-type and
tab/block/section examples.

## Inheritance

Every theme inherits `@Storefront`. To extend another theme, add it to the chain in
each section, just before your own theme:

```javascript
{
  "name": "SwagBasicExampleThemeExtend",
  "views":  ["@Storefront", "@Plugins", "@SwagBasicExampleTheme", "@SwagBasicExampleThemeExtend"],
  "style":  ["app/storefront/src/scss/overrides.scss", "@SwagBasicExampleTheme", "app/storefront/src/scss/base.scss"],
  "script": ["@Storefront", "@SwagBasicExampleTheme", "app/storefront/dist/.../extend.js"],
  "asset":  ["@Storefront", "@SwagBasicExampleTheme", "app/storefront/src/assets"],
  "configInheritance": ["@Storefront", "@SwagBasicExampleTheme"]
}
```

`views`/`style`/`script`/`asset` control template, SCSS, JS, and asset layering;
`configInheritance` makes the parent themes' **config field values** available (shown
with an inherit anchor in the Administration) unless explicitly overridden. Use a base
"corporate design" theme + thin per-sales-channel themes (e.g. a seasonal variant).
Re-run `theme:refresh` to update inheritance relationships after install.
See [references/inheritance.md](references/inheritance.md) for a worked base/extend example.

## SCSS styling

- **`base.scss`** is the entry point for your styles (write normal selectors here).
- **`overrides.scss`** is **only** for overriding Bootstrap/Storefront SCSS variables.
  Bootstrap uses `!default`, so variable overrides must be declared **before** the
  `@Storefront` import — that is why `overrides.scss` sits first in `style`. Example:
  `$border-radius: 0;`. Do **not** put real selectors here.
- Config fields are exposed as SCSS variables under their technical name
  (e.g. `sw-color-brand-primary` → `$sw-color-brand-primary`).

Compile with `bin/console theme:compile`. For live reload during development use the
dev server (`shopware-cli project storefront-watch`, port 9998). Ship JS pre-compiled;
PHP compiles SCSS, webpack handles JS.

### Overriding breakpoints

Six developer-only config fields (since 6.7.8.0) set breakpoints for Twig/JS:
`sw-breakpoint-xs/sm/md/lg/xl/xxl`. To also drive Bootstrap's SCSS grid, reuse them as
the single source of truth in `overrides.scss`:

```scss
$grid-breakpoints: (
  xs: $sw-breakpoint-xs, sm: $sw-breakpoint-sm, md: $sw-breakpoint-md,
  lg: $sw-breakpoint-lg, xl: $sw-breakpoint-xl, xxl: $sw-breakpoint-xxl
);
```

## Assets and icons

Declare asset paths in `theme.json` `asset`. `theme:compile` copies them to
`<shopware root>/public/theme/<theme-asset-uuid>/asset/`. Link them:

```twig
<img src="{{ asset('/assets/your-image.png', 'theme') }}">
```
```scss
body { background-image: url('#{$app-css-relative-asset-path}/your-image.png'); }
```

Icons use the `sw_icon` Twig function. Default pack path:
`<plugin>/src/Resources/app/storefront/dist/assets/icon/default` (other packs → `.../icon/<pack-name>`).
Render with a `namespace` (your theme) and optional `pack`/`size`/`color`/`class`:

```twig
{% sw_icon 'done-outline-24px' style { 'namespace': 'SwagBasicExampleTheme', 'pack': 'solid' } %}
```

Since 6.4.1.0 you can register custom icon locations via the `iconSets` key in
`theme.json` (mandatory when shipping the theme as an app):

```json
{ "iconSets": { "custom-icons": "app/storefront/src/assets/icon-pack/custom-icons" } }
```

Note: icons/custom assets are **not** covered by theme inheritance.

## Troubleshooting

- Theme not visible: `bin/console plugin:refresh` then `plugin:list`.
- Theme not applied: `bin/console theme:change` then `theme:compile`.
- Stale/erroring: `bin/console cache:clear`; check `var/log/` and `custom/plugins/` permissions.
