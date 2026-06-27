# Custom modules — full manifest examples

All `<module>` / `<main-module>` elements live in the `<admin>` section of
`manifest.xml`. Modules render as iframes loaded from their `source` URL.

## Basic module

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/shopware/trunk/src/Core/Framework/App/Manifest/Schema/manifest-3.0.xsd">
    <meta>
        ...
    </meta>
    <admin>
        <module name="exampleModule"
                source="https://example.com/promotion/view/promotion-module"
                parent="sw-marketing"
                position="50">
            <label>Example module</label>
            <label lang="de-DE">Beispiel Modul</label>
        </module>
    </admin>
</manifest>
```

Attributes:

- `name` (required): technical name the module is referenced with.
- `parent`: Administration navigation id of the parent menu item. If omitted the
  module is listed under "My apps" (this menu entry is deprecated; `parent` will
  become required in future versions).
- `source` (optional): URL of the app endpoint serving the module iframe. Omit
  for a module that only serves as a parent for other modules.
- `position` (optional): numeric index controlling order among siblings.
- `<label>` (optionally with `lang`): how the module shows in the admin menu.

## Nesting modules (third menu level)

A module's navigation id follows the pattern `app-<appName>-<moduleName>`. Use
it as the `parent` of another module to group entries:

```xml
<admin>
    <module name="myModules"
            source="https://example.com/promotion/view/promotion-module"
            parent="sw-catalogue"
            position="50">
        <label>My apps modules</label>
        <label lang="de-DE">Module meiner app</label>
    </module>

    <module name="someModule"
            source="https://example.com/promotion/view/promotion-module"
            parent="app-myApp-myModules"
            position="1">
        <label>Module underneath "My apps modules"</label>
        <label lang="de-DE">Modul unterhalb von "Module meiner app"</label>
    </module>
</admin>
```

Parent-only modules do not need `source` (though they may set it).

## Main module

A `<main-module>` opens from the installed-apps list and the app detail page in
the store. Only `source` is required, and it may reuse a regular module's URL.
Kept separate from menu modules to avoid mixing.

```xml
<admin>
    <module name="normalModule"
            source="https://example.com/main"
            parent="sw-catalogue"
            position="50">
        <label>Module in admin menu</label>
    </module>

    <!-- may reuse the same URL -->
    <main-module source="https://example.com/main"/>
</admin>
```

Not compatible with themes — they always open the theme config by default.

## Leaving the loading state

The module iframe must signal that loading finished, otherwise the loading
spinner stays (aborted after ~5 seconds):

```javascript
function sendReadyState() {
    window.parent.postMessage('sw-app-loaded', '*');
}
```

With the Meteor Admin SDK, `location.startAutoResizer()` both signals readiness
and keeps the iframe height in sync with content.

## Determining the requesting shop

When a user opens the module, the app receives a request at the `source` URL
with query parameters identifying the shop:

- `shop-id` — unique identifier of the installing shop.
- `shop-url` — shop URL, usable to call the Shopware API.
- `timestamp` — Unix timestamp of the request.
- `shopware-shop-signature` — SHA256 HMAC of the rest of the query string,
  signed with the `shop-secret` assigned during registration. Verify it to
  authenticate the request.

## Admin design compatibility

Because the module is an iframe, Administration CSS/JS are not available out of
the box. To visually match the Administration, read the shop version from the
`sw-version` query parameter and load the matching compiled Administration
stylesheets, published in tagged releases of the `shopware/administration`
package under `Resources/public/static`.
