# Modules and components — detail

## Module directory layout

```
src/Resources/app/administration/src/
  main.js                         # entry point, auto-loaded by Shopware
  module/swag-example/
    index.js                      # Shopware.Module.register(...)
    page/
      swag-example-list/
      swag-example-detail/
      swag-example-create/
    snippet/
      en-GB.json
      de-DE.json
```

`main.js` must import the module so its `index.js` runs:

```javascript
// main.js
import './module/swag-example';
```

## Full module registration

```javascript
// src/module/swag-example/index.js
import './page/swag-example-list';
import './page/swag-example-detail';
import './page/swag-example-create';
import deDE from './snippet/de-DE';
import enGB from './snippet/en-GB';

Shopware.Module.register('swag-example', {
    type: 'plugin',
    name: 'Example',
    title: 'swag-example.general.mainMenuItemGeneral',
    description: 'swag-example.general.descriptionTextModule',
    color: '#ff3d58',
    icon: 'regular-shopping-bag',

    snippets: {
        'de-DE': deDE,
        'en-GB': enGB,
    },

    routes: {
        list: {
            component: 'swag-example-list',
            path: 'list',
        },
        detail: {
            component: 'swag-example-detail',
            path: 'detail/:id',
            meta: { parentPath: 'swag.example.list' },
        },
        create: {
            component: 'swag-example-create',
            path: 'create',
            meta: { parentPath: 'swag.example.list' },
        },
    },

    navigation: [{
        label: 'swag-example.general.mainMenuItemGeneral',
        color: '#ff3d58',
        path: 'swag.example.list',
        icon: 'regular-shopping-bag',
        position: 100,
    }],
});
```

### Configuration notes

- **`color`** — primary accent used on the module icon, the global-search tag, and
  the smart bar.
- **`icon`** — module icon (see the Shopware icon set / Meteor Icon Kit). The
  navigation menu icon is configured **separately** in the `navigation` entry.
- **`title`** — default browser title; can be overridden per component.
- **`description`** — shown as the empty-state (e.g. an empty list).
- **`type`** — `core` or `plugin`; third-party always uses `plugin`. It is a
  convention, not strictly validated.
- **`name`** — should be technically unique.

## Snippets

Snippet files are nested JSON keyed per locale and prefixed by the extension name.
They are auto-loaded from the `snippet` folder structure.

```json
{
    "swag-example": {
        "general": {
            "mainMenuItemGeneral": "My custom module",
            "descriptionTextModule": "Manage this custom module here"
        }
    }
}
```

Reference keys by path, e.g. `swag-example.general.mainMenuItemGeneral`. Objects can
be nested arbitrarily.

## Settings module item

To list a module under the Administration **Settings** area:

```javascript
Shopware.Module.register('swag-plugin', {
    // ...
    settingsItem: [{
        group: 'plugins',           // 'shop' | 'system' | 'plugins'
        icon: 'regular-rocket',
        to: 'swag.plugin.list',     // a registered route name
        name: 'SwagExampleMenuItemGeneral',  // optional, falls back to module
        label: '',                  // optional, falls back to module
    }],
});
```

## Component registration: async vs sync

### Component placement

- **Page component** (used by a module route):
  `src/module/<module>/page/<component>`
- **Reusable component**: `src/component/<plugin>/<component>`

The path is a recommendation that mirrors core conventions, not a hard requirement.

### Async (preferred — lazy loaded)

Register in `main.js` with a dynamic import; export the config from `index.js`.

```javascript
// main.js
Shopware.Component.register('hello-world', () => import('./component/custom-component/hello-world'));
```

```javascript
// component/custom-component/hello-world/index.js
import template from './hello-world.html.twig';

export default Shopware.Component.wrapComponentConfig({
    template,
});
```

`wrapComponentConfig` provides full type support for the config object.

### Sync

Register directly in the component's `index.js`, and import it from `main.js`.

```javascript
// main.js
import './component/custom-component/hello-world';
```

```javascript
// component/custom-component/hello-world/index.js
import template from './hello-world.html.twig';

Shopware.Component.register('hello-world', {
    template,
});
```

Synchronous loading usually just slows the Administration boot; prefer async.

### Templates

Define templates in separate `<component>.html.twig` files and `import` them. Inline
string templates (`template: '<h2>Hello world!</h2>'`) are only for trivial cases.

## Building

```bash
# Template / shopware-cli setup
shopware-cli project admin-build

# platform-only (contribution) setup
composer run build:js:admin
```

The plugin must be active. The bundled JS is emitted to the plugin's
`Resources/public/administration/js/<plugin-name>.js` and copied into the Shopware
`public/bundles/...` directory; include it when publishing.
