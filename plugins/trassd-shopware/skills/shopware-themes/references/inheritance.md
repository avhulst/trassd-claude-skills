# Theme inheritance — worked example

Pattern: a base "corporate design" theme (`SwagBasicExampleTheme`) plus a thin extending
theme (`SwagBasicExampleThemeExtend`) for a seasonal/per-sales-channel variant.

Every theme always inherits `@Storefront`. The placeholder appears in `views`, `style`,
`script`, and `asset` of a fresh theme. To inherit from another theme, add it to each
chain **before** your own theme.

## Base theme config (SwagBasicExampleTheme)

```javascript
{
  "name": "SwagBasicExampleTheme",
  "config": {
    "fields": {
      "sw-color-brand-primary": {
        "type": "color", "value": "#399", "editable": true,
        "tab": "colors", "block": "themeColors", "section": "importantColors"
      },
      "sw-brand-icon": { "type": "url", "value": "/our-logo.png", "editable": true }
    }
  }
}
```

## Extending theme (SwagBasicExampleThemeExtend)

```javascript
{
  "name": "SwagBasicExampleThemeExtend",
  "author": "Shopware AG",
  "views":  ["@Storefront", "@Plugins", "@SwagBasicExampleTheme", "@SwagBasicExampleThemeExtend"],
  "style":  ["app/storefront/src/scss/overrides.scss", "@SwagBasicExampleTheme", "app/storefront/src/scss/base.scss"],
  "script": ["@Storefront", "@SwagBasicExampleTheme", "app/storefront/dist/storefront/js/swag-example-plugin-theme-extended/swag-example-plugin-theme-extended.js"],
  "asset":  ["@Storefront", "@SwagBasicExampleTheme", "app/storefront/src/assets"],
  "configInheritance": ["@Storefront", "@SwagBasicExampleTheme"],
  "config": {
    "fields": {
      "sw-brand-icon": { "type": "url", "value": "/our-logo-holidays.png", "editable": true },
      "sw-advent-calendar-background-color": { "type": "color", "value": "#399", "editable": true }
    }
  }
}
```

## How each section layers

- **`views`** — render order: Storefront base → installed plugins → `@SwagBasicExampleTheme`
  → current theme. Later entries override earlier ones.
- **`script`** — same layering for JavaScript. Reference **compiled** `dist/` files.
- **`style`** — same idea for SCSS. `overrides.scss` stays first because variable
  overrides (e.g. `$border-radius`) must precede the imports that use them.
- **`asset`** — list a parent theme here only if you reuse its assets. (Icons and custom
  assets are **not** inherited.)
- **`configInheritance`** — inherits the **field config values** from the listed themes.
  The last listed theme that differs from the current one becomes the parent. `Storefront`
  is always inherited even without this key.

In the example, inherited fields appear in the Administration with an inherit anchor and
use the parent value until changed. `sw-brand-icon` is given a new default here, so it is
**not** inherited; `sw-advent-calendar-background-color` is a brand-new field for the
seasonal variant.

The inheritance relationship is established at install time — run
`bin/console theme:refresh` to update it afterward.
