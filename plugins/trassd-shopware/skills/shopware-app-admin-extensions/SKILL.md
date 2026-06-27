---
name: shopware-app-admin-extensions
description: >-
  Build Shopware 6 Administration extensions for Apps with the Meteor Admin SDK
  — custom modules, action buttons, CMS elements, and snippets loaded from an
  external app. Covers the iframe/postMessage model, manifest.xml <admin>
  declarations (<module>, <main-module>, <action-button>), SDK installation and
  base setup, registering CMS blocks/elements via the cms API, and adding
  Administration snippets. Triggers when creating an admin extension for a
  Shopware app, wiring up the Meteor Admin SDK, adding an app module/action
  button/CMS element to the Administration, or adding admin translations.
---

# Shopware App Administration Extensions

Apps cannot freely override or extend Administration components. JS files placed
in an app's `Resources/administration` namespace are **ignored**. Instead, apps
extend the Administration through a small set of defined extension points:

- **Custom modules** and a **main module** (declared in `manifest.xml` `<admin>`)
- **Action buttons** (declared in `manifest.xml` `<admin>`)
- **CMS blocks/elements** (registered at runtime via the Meteor Admin SDK)
- **Snippets/translations** (JSON files under `Resources/app/administration/snippet`)

## The model: iframe + Meteor Admin SDK

An app module renders as an **iframe** embedded in the Administration. The app
serves an external frontend at the module's `source` URL; that frontend talks to
the Administration over **postMessage**, abstracted by the
**Meteor Admin SDK** (`@shopware-ag/meteor-admin-sdk`).

- The SDK works identically for apps and plugins; it provides helpers for
  notifications, context, data subscriptions, and UI extension.
- For simple apps, `manifest.xml` declarations alone (modules, action buttons)
  are enough. For advanced behavior (CMS elements, live data sync, UI locations)
  use the SDK.
- Because the module is an iframe, the parent Administration cannot tell when
  the app has finished loading. Signal readiness to dismiss the loading spinner
  (aborted after ~5s if not signaled):

  ```javascript
  function sendReadyState() {
      window.parent.postMessage('sw-app-loaded', '*');
  }
  ```

  Using the SDK, the equivalent + auto-resize is `location.startAutoResizer()`.

### Install and base setup

For a local Admin UI dev loop, the app is defined by `manifest.xml`, discovered
from `custom/apps`, and the module loads from a dev server (e.g. Vite on
`5173`). Install the SDK in the frontend project:

```bash
npm install @shopware-ag/meteor-admin-sdk
```

(The local-dev guide also references `@shopware-ag/admin-extension-sdk`; prefer
the Meteor Admin SDK for new work.)

`<meta>` requires `<author>` and `<copyright>` — `bin/console app:refresh`
fails if either is missing. Register/activate the app:

```bash
bin/console app:refresh
bin/console app:install --activate MyAdminTestApp
```

Authenticating iframe requests (signatures, `shop-id`/`shop-url`/`timestamp`/
`shopware-shop-signature` query params, app backend setup) is out of scope here
— see the app registration and webhook guides referenced by the docs.

## Custom modules

Declare modules in `manifest.xml` `<admin>`. Add any number of `<module>`
elements. See [references/modules.md](references/modules.md) for full examples
(menu nesting, main module, design compatibility).

```xml
<admin>
    <module name="exampleModule"
            source="https://example.com/promotion/view/promotion-module"
            parent="sw-marketing"
            position="50">
        <label>Example module</label>
        <label lang="de-DE">Beispiel Modul</label>
    </module>
</admin>
```

Key attributes:

- `name` (required): technical name the module is referenced by.
- `parent`: Administration navigation id of the parent menu item. Omitting it
  lists the module under "My apps" (deprecated; `parent` will become required).
- `source` (optional): app endpoint URL serving the iframe. Omit it for a module
  that only acts as a parent for other modules.
- `position` (optional): numeric sort index among siblings.
- `<label>` (optionally per `lang`): menu display name.

A module's navigation id follows `app-<appName>-<moduleName>`, which other
modules can target as their `parent` to build a submenu.

**Main module** — a single `<main-module source="..."/>` opens from the app's
detail/listing page in the store/My Extensions. Only `source` is required; it
may reuse a regular module's URL. Not compatible with themes (they open theme
config).

## Action buttons

Add buttons to the smartbar of `detail` and `list` views via `<action-button>`
under `<admin>`:

```xml
<admin>
    <action-button action="restockProduct" entity="product" view="list"
                   url="https://example.com/restock">
        <label>restock</label>
    </action-button>
</admin>
```

Attributes: `action` (free unique id), `entity` (entity worked on), `view`
(`detail` or `list`), `url` (target). On click the app receives a webhook-like
signed request containing `entity`, `action`, and the selected `ids` (a single
id on detail views). Verify it via `shopware-shop-signature`.

The response (with `shopware-app-signature` header) can trigger an
Administration action via `actionType`: `notification`, `reload`, `openNewTab`,
or `openModal`. A relative `url` (e.g. `/api/script/action-button`) targets an
app-script custom endpoint instead of an external server. See
[references/action-buttons.md](references/action-buttons.md) for response
payloads and the app-script variant.

## CMS elements via the Admin SDK

Adding a CMS element requires the Meteor Admin SDK (the declarative `cms.xml`
alternative only reuses existing elements inside block slots). The app file
structure is loaded entirely via iframe; Vue 3 SFCs are recommended.

The Administration drives the app through **location IDs**. The entry point
branches on the current location:

```javascript
import { location } from '@shopware-ag/meteor-admin-sdk';

if (location.is(location.MAIN_HIDDEN)) {
    import('./base/mainCommands'); // logic only — register blocks/elements here
} else {
    import('./viewRenderer');      // render the Vue view for location.get()
}
```

Register both a block (selectable in the block picker) and an element (fills the
slot) in the `MAIN_HIDDEN` branch:

```javascript
import { cms } from '@shopware-ag/meteor-admin-sdk';

void cms.registerCmsBlock({ name: 'swag-dailymotion', label: 'Dailymotion video',
    category: 'video', slots: [{ element: 'swag-dailymotion' }] });

void cms.registerCmsElement({ name: 'swag-dailymotion', label: 'Dailymotion video',
    defaultConfig: { dailyUrl: { source: 'static', value: '' } } });
```

- `registerCmsElement` only → appears in the element-replacement modal.
- `registerCmsBlock` only → appears in the block picker but renders nothing.
- Both → fully discoverable and functional.

Each element auto-generates three location IDs from its name plus `-element`,
`-config`, and `-preview`; map each to a Vue component in `viewRenderer.ts`.
Data flows over the SDK `data` API (`data.get`/`data.update`/`data.subscribe`),
keyed by a **publishing key** = element name + `__config-element`, combined with
the `elementId` query param Shopware appends to the iframe URL. A Storefront
template at `<app>/Resources/views/storefront/element/<name>.html.twig` renders
the saved config. Full `viewRenderer`, config/element/preview SFCs, and the
publishing-key/`dataId` pattern are in
[references/cms-elements.md](references/cms-elements.md).

## Snippets / translations

Place admin snippet JSON files under
`<app root>/Resources/app/administration/snippet`, one per locale (`en.json`,
`de.json`), with optional dialect patch files (`en-US.json`). Apps **may not**
override existing Shopware snippet keys — only add new ones. Structure and
fallback-language behavior otherwise match plugin snippets.

## Quick checklist

- Module/action-button/main-module declared under `<admin>` in `manifest.xml`;
  `<meta>` has `<author>` + `<copyright>`.
- App refreshed/installed/activated via `bin/console app:*`.
- Module iframe signals readiness (`sw-app-loaded` / `startAutoResizer`).
- Incoming requests (modules, action buttons) verified via
  `shopware-shop-signature`; responses signed with `shopware-app-signature`.
- CMS additions register both block and element; publishing key uses the
  `__config-element` suffix.
- Snippet files add new keys only, per locale.
