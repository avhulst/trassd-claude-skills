---
name: shopware-administration
description: >-
  Extend the Shopware 6 Administration (the Vue.js SPA backend) — register custom
  modules with routes/navigation/snippets, register custom components with
  twig-based templates, override or extend core components (Component.override /
  Component.extend with the $super call), and reuse sw-* base components instead of
  raw HTML. Trigger when adding or changing an admin module/component under a
  plugin's Resources/app/administration directory, or when customizing the admin UI.
---

# Shopware Administration

The Administration is Shopware 6's backend single-page app built on Vue.js. You
extend it from a plugin, never by editing core. The bridge to the SPA is a single
global object, `Shopware`, that exposes factories like `Shopware.Module` and
`Shopware.Component`.

## Critical conventions

- **Entry point is `main.js`.** Shopware automatically loads
  `<plugin root>/src/Resources/app/administration/src/main.js`. Everything you add
  must be imported (directly or transitively) from there.
- **No Vue SFCs.** Components are plain JS config objects registered via the global
  `Shopware` factories; templates are **TwigJS** `.html.twig` files, not `<template>`
  blocks in `.vue` files. Only TwigJS's `block` system is used (extending/overriding
  blocks) — no other Twig features.
- **Use `Shopware.*` factories, not the internal factories directly.**
  `Shopware.Module.register()` wraps the `ModuleFactory`;
  `Shopware.Component.register()` / `.override()` / `.extend()` wrap the
  `ComponentFactory`.
- **Translation keys, not literals.** Module `title`, `description`, and navigation
  `label` expect i18n keys (vue-i18n), resolved from `snippets`.
- **Build before testing.** Run `shopware-cli project admin-build` (or
  `composer run build:js:admin` in a contribution setup); the plugin must be active.

## Registering a module

A module groups routes, navigation, snippets, and metadata. Place it under
`src/module/<name>/index.js` and import it from `main.js`
(`import './module/swag-example';`).

```javascript
// src/module/swag-example/index.js
import enGB from './snippet/en-GB';
import deDE from './snippet/de-DE';

Shopware.Module.register('swag-example', {
    type: 'plugin',                 // third-party modules use 'plugin'
    name: 'Example',                // technically unique
    title: 'swag-example.general.mainMenuItemGeneral',   // i18n key
    description: 'swag-example.general.descriptionTextModule',
    color: '#ff3d58',               // accent color
    icon: 'regular-shopping-bag',   // module icon

    snippets: { 'de-DE': deDE, 'en-GB': enGB },

    routes: {
        list:   { component: 'swag-example-list', path: 'list' },
        detail: { component: 'swag-example-detail', path: 'detail/:id',
                  meta: { parentPath: 'swag.example.list' } },
    },

    navigation: [{
        label: 'swag-example.general.mainMenuItemGeneral',
        path: 'swag.example.list',
        icon: 'regular-shopping-bag',
        position: 100,
    }],
});
```

- `type: 'plugin'` is convention; the factory only special-cases `plugin` vs `core`.
- Route `component` values reference registered component names (the page
  components). `parentPath` wires breadcrumbs/back navigation.
- Snippet objects are nested per locale and prefixed by the extension name; store
  them as `src/module/<name>/snippet/en-GB.json` and import them.
- To surface a module under **Settings**, add a `settingsItem` array
  (`group` of `shop` | `system` | `plugins`, plus `icon` and `to` = route name).

## Registering a component

Decide placement first: page components for routes go under
`src/module/<module>/page/<component>`; reusable components go under
`src/component/<plugin>/<component>`. Register either asynchronously (preferred,
lazy-loaded) or synchronously.

```javascript
// Async (preferred) — register in main.js, export config from index.js
Shopware.Component.register('hello-world', () => import('./component/custom-component/hello-world'));
```

```javascript
// index.js (async): export the config, wrapped for type support
import template from './hello-world.html.twig';

export default Shopware.Component.wrapComponentConfig({
    template,        // shorthand for template: template
});
```

For synchronous loading, call `Shopware.Component.register('hello-world', { template })`
directly in the component's `index.js` and `import './component/.../hello-world';`
from `main.js`. Once registered, use the component anywhere:
`<hello-world></hello-world>`.

Keep templates in separate `.html.twig` files and `import` them — inline string
templates are only for trivial cases.

## Overriding vs. extending core components

- **`Shopware.Component.override(name, config)`** — replace behavior of an existing
  component **in place** (same name). Use to tweak a core component everywhere.
- **`Shopware.Component.extend(newName, baseName, config)`** — create a **new**
  component derived from a base, leaving the base untouched.

```javascript
// Override the dashboard's template
import template from './sw-dashboard-index.html.twig';

Shopware.Component.override('sw-dashboard-index', { template });
```

```javascript
// Extend sw-text-field into a new sw-custom-field
import template from './sw-custom-field.html.twig';

Shopware.Component.extend('sw-custom-field', 'sw-text-field', { template });
```

When overriding/extending **methods or computed properties**, call the original
implementation with `this.$super('<name>', ...args)`; omit it to replace entirely:

```javascript
Shopware.Component.extend('sw-custom-field', 'sw-text-field', {
    methods: {
        onInput() {
            const result = this.$super('onInput');  // run original
            // add custom logic
        },
    },
});
```

In override/extend templates, use TwigJS blocks: redefine a `{% block %}` to replace
it, and include `{% parent %}` to keep the original markup. Override the smallest
block that achieves the goal (find the block name in the core component's
`.html.twig`). Remember to import the override's `index.js` from `main.js`.

A newer **experimental Composition API extension system**
(`Shopware.Component.overrideComponentSetup`, `createExtendableSetup`) exists for
components written with the Composition API. Keep using the Options-API
factory (`override`/`extend`) for Options-API components; only reach for the
Composition-API system when the target component uses it. See
[references/extending-components.md](references/extending-components.md).

## Use base (`sw-*`) components

The Administration ships a large library of registered Vue components, available
globally in every template — use them instead of raw HTML so you inherit styling,
inheritance handling, slots, and props.

```twig
<div>
    <sw-text-field />
</div>
```

Browse available components, their props, and slots in the Shopware Component
Library. Prefer `sw-text-field`, `sw-card`, `sw-button`, etc., over hand-rolled
markup.

## More detail

- [references/module-and-component.md](references/module-and-component.md) — full
  module config, snippet file layout, settings items, sync vs. async component
  registration.
- [references/extending-components.md](references/extending-components.md) — block
  overriding walkthrough, `$super` for computed properties, and the experimental
  Composition API extension system.
