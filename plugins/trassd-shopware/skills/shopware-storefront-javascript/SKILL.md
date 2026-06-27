---
name: shopware-storefront-javascript
description: >-
  Build interactive Storefront components with the JavaScript plugin system —
  the Plugin base class, registering plugins with the PluginManager, binding via
  data-* attributes & options, the event/HttpClient helpers, overriding existing
  JS plugins, and bundling the JS. Triggers when adding or overriding storefront
  JavaScript under Resources/app/storefront/src or wiring a data-bound storefront
  component.
---

# Shopware Storefront JavaScript

How to write, register, configure, and override interactive Storefront
components — the client-side JS classes that the `PluginManager` binds to DOM
elements. This is the **JavaScript** side of the Storefront and is distinct from
PHP controllers/pages/Twig (see the `shopware-storefront` skill for those).

## When this applies

You are adding interactivity under
`<plugin root>/src/Resources/app/storefront/src/`, binding a JS class to DOM
elements via `data-*` attributes, reacting to another plugin's events, fetching
data client-side, or overriding/extending a built-in Storefront JS plugin.

## Mental model

- A Storefront JS plugin is a **vanilla ES6 class** that extends the Storefront
  `Plugin` base class. It is not a Vue/Symfony plugin.
- The **`PluginManager`** (on `window.PluginManager`) holds the registry. It
  matches registered selectors against the DOM and instantiates one plugin
  **instance per matching element** on `DOMContentLoaded`.
- Each instance gets `this.el` (the bound element), `this.options` (merged
  defaults + per-element JSON options) and its own event emitter `this.$emitter`.

## Entry point and registration

Shopware auto-loads `<plugin root>/src/Resources/app/storefront/src/main.js` as
the Storefront JS entry point. Register plugins there via `PluginManager`:

```javascript
// .../src/Resources/app/storefront/src/main.js
import ExamplePlugin from './example-plugin/example-plugin.plugin';

const PluginManager = window.PluginManager;

// register(name, pluginClass, selector?, options?)
PluginManager.register('ExamplePlugin', ExamplePlugin, '[data-example-plugin]');
```

Rules:

- **Always bind to a selector.** Without a selector the plugin is global; with
  `'[data-example-plugin]'` it runs once per matching element and `this.el`
  points at that element. Prefer a `data-*` attribute selector.
- The registration `name` is PascalCase (`'ExamplePlugin'`); the convention is
  that the DOM attribute is its kebab-case form (`data-example-plugin`).
- **Async / on-demand plugins:** pass a dynamic import factory instead of an
  imported class:
  `PluginManager.register('ExamplePlugin', () => import('./.../example-plugin.plugin'), '[data-example-plugin]');`.
  The PluginManager treats it as async automatically and excludes it from the
  main `storefront.js` bundle — it is fetched only when the selector is present
  on the page. Use this for plugins not needed on every page.

## Writing a Plugin subclass

```javascript
// .../src/example-plugin/example-plugin.plugin.js
const { PluginBaseClass } = window;

export default class ExamplePlugin extends PluginBaseClass {
    static options = {
        text: 'Default text',
    };

    init() {
        this._registerEvents();
    }

    _registerEvents() {
        this.el.addEventListener('click', this.onClick.bind(this));
    }

    onClick() {
        alert(this.options.text);
    }
}
```

- Get the base class from `const { PluginBaseClass } = window;` (or
  `import Plugin from 'src/plugin-system/plugin.class'` in a contribution
  setup). Both refer to the same Storefront `Plugin` base class.
- **`init()` is mandatory** — it is the entry point for your logic and runs
  after the DOM is loaded. (The base `init()` only warns; you must override it.)
- `this.el` — the bound DOM element. `this.options` — the merged options object.
  `this.$emitter` — this instance's `NativeEventEmitter` (see events below).
- `_registerEvents()` is a **convention**, not a base-class method: do your
  `addEventListener` wiring there and call it from `init()`. Bind handlers
  (`.bind(this)`) so `this` stays the instance.
- An optional `update()` method is called by the PluginManager when the plugin
  is re-initialized (e.g. after AJAX replaces part of the DOM).

## Configuring via data-options

Define a `static options = {}` with defaults on the class, then override per
element from Twig using `data-<plugin-name>-options` with a JSON value:

```twig
{% set examplePluginOptions = { text: "Are you not interested?" } %}

<template data-example-plugin
          data-example-plugin-options='{{ examplePluginOptions|json_encode }}'></template>
```

The PluginManager deep-merges `static options` ← previously merged options ←
the per-element JSON into `this.options`. Best practice: build the options into
a Twig variable (set then pass) so other plugins can extend it with
`replace_recursive`, rather than inlining the JSON.

## Binding to the DOM (Twig)

Add an element carrying the plugin's `data-*` attribute on the pages where the
plugin should run, by extending the relevant template block:

```twig
{% sw_extends '@Storefront/storefront/page/content/index.html.twig' %}
{% block base_main_inner %}
    {{ parent() }}
    <template data-example-plugin></template>
{% endblock %}
```

## Reacting to events

Every plugin instance has its **own** `$emitter`, so you cannot subscribe to
another plugin's events on your own emitter. Fetch the target instance from the
PluginManager and subscribe to **its** emitter:

```javascript
init() {
    const el = document.querySelector('[data-cookie-permission]');
    const plugin = window.PluginManager.getPluginInstanceFromElement(el, 'CookiePermission');
    plugin.$emitter.subscribe('hideCookieBar', this.onHideCookieBar);
}
```

- Discover events by searching the core for `this.$emitter.publish(...)` under
  `Resources/app/storefront/src`.
- These events are **notifications** — subscribing does not prevent the original
  behavior. If you need to change behavior, override the plugin instead.
- You can also use plain native DOM events (`addEventListener`) and Shopware's
  `DomAccess` helper for safer DOM access.

## Overriding an existing plugin

Extend the core class, then register with `override()` instead of `register()`:

```javascript
// override class
import CookiePermissionPlugin from 'src/plugin/cookie/cookie-permission.plugin';

export default class MyCookiePermission extends CookiePermissionPlugin {
    init() {
        // change behavior, then call the parent
        super.init();
    }
}
```

```javascript
// main.js — override(name, pluginClass, selector?)
PluginManager.override('CookiePermission', MyCookiePermission, '[data-cookie-permission]');
```

- Call `super.<method>()` to keep the original behavior where wanted.
- If you cannot import the class (e.g. a third-party plugin with no alias), get
  it at runtime:
  `const PluginClass = window.PluginManager.getPlugin('CookiePermission').get('class');`
  then extend `PluginClass`.
- **A plugin can be overridden only once** — the last override wins. If only a
  notification is needed, prefer subscribing to an event.
- Overriding an async plugin requires an async (dynamic-import) override.
- `override()` is effectively `deregister()` + `register()` for the same name;
  see [references/overriding.md](references/overriding.md).

## Fetching data

Use the native `fetch` API for AJAX. Shopware also ships an `HttpClient` service
helper (`get`, `post`, `abort`) for callback-style requests.

```javascript
async fetchData() {
    const response = await fetch('/widgets/checkout/info');
    const data = await response.text();
}
```

See [references/fetching-and-http-client.md](references/fetching-and-http-client.md).

## Script tag alternative

When you need an external/CDN script rather than bundled JS, extend the head
template block instead of registering a plugin:

```twig
{% sw_extends '@Storefront/storefront/layout/meta.html.twig' %}
{% block layout_head_javascript_hmr_mode %}
    {{ parent() }}  {# MUST keep — renders the Storefront JS #}
    <script src="https://unpkg.com/isotope-layout@3/dist/isotope.pkgd.min.js" defer></script>
{% endblock %}
```

Always call `{{ parent() }}` (omitting it disables core Storefront JS). Prefer
`defer` (or `async` if the library requires it). See
[references/script-tag.md](references/script-tag.md).

## Building the Storefront JS

Source lives in `.../app/storefront/src/`; the compiled output is written to
`.../app/storefront/dist/storefront/js/<plugin-name>/<plugin-name>.js` and is
picked up automatically. **Ship the compiled `dist/` file with your plugin.**

```bash
shopware-cli project storefront-build        # template / project setup
composer run build:js:storefront             # platform contribution setup
```

For local development use the storefront watcher / hot-proxy to recompile on
change instead of a full build.

## Built-in plugins & helpers

Before writing new JS, check whether a built-in plugin already does it
(`AddToCartPlugin`, `ListingPlugin`, `OffCanvasCartPlugin`,
`CookiePermissionPlugin`, sliders, forms, …) and the helpers (`DomAccess`,
`CookieStorage`, `HttpClient`, `Iterator`, `DeviceDetection`,
`ViewportDetection`). The full catalogue is the Storefront Plugins and Helper
Reference in the official docs.

## Checklist

- [ ] Class extends the Storefront `Plugin` base class and implements `init()`.
- [ ] Registered in `main.js` via `PluginManager.register(...)` with a `data-*`
      selector (or `override(...)` for overrides).
- [ ] Defaults in `static options`; per-element overrides via
      `data-<name>-options` JSON.
- [ ] Cross-plugin events subscribed via the target instance's `$emitter`.
- [ ] Override used only when an event won't do; `super` called as needed.
- [ ] Storefront built and the compiled `dist/` file shipped.
