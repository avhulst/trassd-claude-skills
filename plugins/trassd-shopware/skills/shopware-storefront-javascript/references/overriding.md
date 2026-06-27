# Overriding & extending an existing Storefront JS plugin

Storefront JS plugins are vanilla ES6 classes, so you override one by extending
its class and re-registering it under the same name with `PluginManager.override()`.

## Directory & class

Create `<plugin root>/src/Resources/app/storefront/src/my-cookie-permission/my-cookie-permission.plugin.js`
and extend the core class:

```javascript
import CookiePermissionPlugin from 'src/plugin/cookie/cookie-permission.plugin';
import CookieStorage from 'src/helper/storage/cookie-storage.helper';

export default class MyCookiePermission extends CookiePermissionPlugin {
    init() {
        // Force the cookie bar to always show by clearing the stored preference
        CookieStorage.setItem(this.options.cookieName, '');
        super.init();
    }

    _hideCookieBar() {
        if (confirm('Do you want to hide the cookie bar?')) {
            super._hideCookieBar();
        }
    }
}
```

- Override individual methods and call `super.<method>()` to retain the original
  behavior where you still want it.
- `this.options` is inherited from the parent; reuse its option names
  (e.g. `this.options.cookieName`) rather than redefining them.

## Getting the class without an import

If the original plugin cannot be imported directly (e.g. a third-party plugin
with no module alias), fetch the registered class at runtime:

```javascript
const PluginManager = window.PluginManager;
const plugin = PluginManager.getPlugin('CookiePermission');
const PluginClass = plugin.get('class');

export default class MyCookiePermission extends PluginClass {
    // ...
}
```

## Registering the override

In `main.js`, use `override()` (not `register()`):

```javascript
import MyCookiePermission from './my-cookie-permission/my-cookie-permission.plugin';

const PluginManager = window.PluginManager;
PluginManager.override('CookiePermission', MyCookiePermission, '[data-cookie-permission]');
```

For an async core plugin, the override must be async too:

```javascript
PluginManager.override(
    'CookiePermission',
    () => import('./my-cookie-permission/my-cookie-permission.plugin'),
    '[data-cookie-permission]'
);
```

## Constraints

- **Each plugin can be overridden only once.** If two Shopware plugins override
  the same one, only the last registered override is effective.
- `override()` behaves like `deregister()` followed by `register()` for the same
  name (the PluginManager also exposes `deregister(name, selector)` directly).
- When you only need to *react* to behavior (not change it), subscribe to a
  published event instead — see the events section of the SKILL.

## Build & verify

```bash
shopware-cli project storefront-build     # or: composer run build:js:storefront
```

Reload the Storefront and confirm the overridden behavior.
