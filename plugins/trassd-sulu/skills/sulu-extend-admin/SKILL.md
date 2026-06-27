---
name: sulu-extend-admin
description: >-
  Extend the Sulu administration interface — Admin classes, navigation items,
  view builders, form/list metadata, and the JS admin bundle build. Use when
  adding admin tabs or views, registering an Admin service, configuring list or
  form metadata for an entity, wiring a resource, or building/updating the
  admin frontend (webpack/Encore, sulu:admin:update-build).
---

# Extending the Sulu admin UI

Sulu's admin is a React single-page app. You integrate a custom entity by
declaring REST routes, registering a **resource**, and adding **views** +
**navigation items** from a PHP **`Admin`** class — usually without writing any
JavaScript. The frontend's predefined views (list, form, resource-tabs) are
configured through PHP view builders plus XML metadata.

## The pipeline (do these in order)

1. Expose a standard REST API for the entity (`cget`/`get`/`post`/`put`/`delete`).
2. Register list/detail routes under a **resource key** in `sulu_admin.resources`.
3. Write **list** metadata (`config/lists/<key>.xml`) and **form** metadata
   (`config/forms/<key>.xml`).
4. Add an `Admin` class that registers views and navigation items.
5. Rebuild the admin frontend so changes are picked up.

## REST API + resource registration

- The admin expects standard REST actions behind one collection URL
  (`/admin/api/events`) and one item URL (`/admin/api/events/{id}`). Implement
  only the actions you actually need.
- Add a unique `RESOURCE_KEY` constant on the entity; reuse it as list key
  unless you need multiple lists per entity.
- For list responses use the `ListBuilder` abstraction (`FieldDescriptorFactory`
  + `DoctrineListBuilderFactory` + `RestHelper` + `PaginatedRepresentation`) so
  pagination/search/sort match what the list view expects.
- Map the routes to the resource key:

  ```yaml
  # config/packages/sulu_admin.yaml
  sulu_admin:
      resources:
          events:
              routes:
                  list: app.get_events     # collection route name
                  detail: app.get_event    # item route name (must include the ID)
  ```

- To expose routes to the frontend, use `FOSJsRoutingBundle` (`expose: true`).
  Sulu auto-exposes route names matching `(.+\.)?c?get_.*`.

See [references/api-and-resources.md](references/api-and-resources.md) for a full
controller, route YAML, and the list/form metadata XML.

## The Admin class

Extend `Sulu\Bundle\AdminBundle\Admin\Admin` and override the two hook methods.
Symfony autoconfigure registers it automatically — no manual service config
needed in app projects. The two hooks are `configureViews(ViewCollection)` and
`configureNavigationItems(NavigationItemCollection)`. Inject
`ViewBuilderFactoryInterface` (and any other services you need, e.g. a webspace
manager or security checker).

```php
use Sulu\Bundle\AdminBundle\Admin\Admin;
use Sulu\Bundle\AdminBundle\Admin\Navigation\NavigationItemCollection;
use Sulu\Bundle\AdminBundle\Admin\View\ViewBuilderFactoryInterface;
use Sulu\Bundle\AdminBundle\Admin\View\ViewCollection;

class EventAdmin extends Admin
{
    public function __construct(private ViewBuilderFactoryInterface $viewBuilderFactory) {}

    public function configureViews(ViewCollection $viewCollection): void { /* ... */ }
    public function configureNavigationItems(NavigationItemCollection $c): void { /* ... */ }
}
```

- If you register the service explicitly (e.g. in a bundle), tag it
  `sulu.admin` plus `{ name: 'sulu.context', context: 'admin' }`. Verify with
  `php bin/console debug:container --tag=sulu.admin`.
- Admin classes are services collected via tags, so they may depend on other
  services — adding to the admin runs in normal DI, not in a static config.

## Navigation items

`NavigationItem` takes a name (also used as translation key). Point it at a view
by name, set an icon, and order it with a position.

```php
$item = new NavigationItem('app.events');
$item->setView(static::EVENT_LIST_VIEW);  // a view name registered in configureViews
$item->setIcon('su-calendar');            // su-* = Sulu icon font, fa-* = Font Awesome
$item->setPosition(30);                   // items sort by position
$navigationItemCollection->add($item);
```

## Views via view builders

A `View` needs a unique `name`, a `path` (URL), and a `type` (the React
component key in the `ViewRegistry`). Never build `View` directly — use
`ViewBuilderFactoryInterface`, which returns a typed, validating builder per
view type:

- `createListViewBuilder($name, $path)` — list of a resource.
- `createFormViewBuilder($name, $path)` — a form (one tab).
- `createResourceTabViewBuilder($name, $path)` — parent that loads the resource
  once and hosts child form tabs.
- `createPreviewFormViewBuilder($name, $path)` — form tab with live preview.
- `createFormOverlayListViewBuilder($name, $path)` — list editing rows in an overlay.

Always `$viewCollection->add($builder)`. Adding to the collection (rather than
returning) lets later bundles manipulate views added earlier.

Minimal list view:

```php
$viewCollection->add(
    $this->viewBuilderFactory->createListViewBuilder(static::EVENT_LIST_VIEW, '/events')
        ->setResourceKey(Event::RESOURCE_KEY)
        ->setListKey('events')         // matches <key> in config/lists/events.xml
        ->addListAdapters(['table'])
);
```

Inspect registered views with `bin/adminconsole sulu:admin:debug-view`.

For the full add/edit form flow (list linked to add/edit views via `setAddView`
/ `setEditView`, `ResourceTabs` parents, `setParent`, `ToolbarAction`s), see
[references/views-and-forms.md](references/views-and-forms.md).

## List and form metadata

The XML supplies the rendering info Doctrine mapping cannot. Translations come
from the `admin` translation domain (e.g. `admin.en.json`).

- **List** (`config/lists/<key>.xml`): root `<list>`, a unique `<key>`, then
  `<property>` entries with `name`, `visibility`
  (`yes`/`no`/`always`/`never`), `searchability`, `type` (e.g. `datetime`),
  `translation`, plus `<field-name>` / `<entity-name>` so the `ListBuilder` can
  build an efficient query.
- **Form** (`config/forms/<key>.xml`): root `<form>`, a unique `<key>` (usually
  resourceKey + suffix), then `<property>` with `name`, `type` (a field type
  from the `fieldRegistry`, e.g. `text_line`, `date`, `single_media_selection`),
  `mandatory`, `colspan` (1–12), optional `visibleCondition`/`disabledCondition`,
  a `<meta><title>` translation key, and `<params>`.

`resourceKey` loads the data; `formKey` selects which fields to show — splitting
them lets one resource span several form tabs.

Custom selection fields (relations to other entities) need no JS: register them
under `sulu_admin.field_type_options` (`selection` / `single_selection`
blueprints) and reference the new type in the form XML. See
[references/api-and-resources.md](references/api-and-resources.md).

## Adding a tab to an existing resource (e.g. pages)

Add a child form view whose `setParent(...)` is the existing edit-form view
constant — e.g. `PageAdmin::EDIT_FORM_VIEW` — and it appears as a new tab.

```php
$viewCollection->add(
    $this->viewBuilderFactory
        ->createPreviewFormViewBuilder('sulu_page.page_edit_form.socials', '/socials')
        ->setResourceKey('pages')
        ->setFormKey('page_socials')      // config/forms/page_socials.xml
        ->setTabTitle('Socials')
        ->addToolbarActions([new ToolbarAction('sulu_admin.save_with_publishing')])
        ->setParent(PageAdmin::EDIT_FORM_VIEW)
);
```

- Order tabs with `setTabOrder` / `setTabPriority`.
- The child path (`/socials`) is appended to the parent path automatically.
- Use slashes in property names (`ext/social/twitter_title`) to nest values.
- Gate the tab with a permission check (inject the security checker / webspace
  manager) when the parent resource is permission-controlled.
- Showing the tab is separate from persisting its data — for fields outside
  template data you must handle storage (entity extension, content
  merger/mapper/normalizer). That persistence layer is out of scope here.

Full tab example: [references/views-and-forms.md](references/views-and-forms.md).

## Building the admin frontend

After updating Sulu or adding custom JS, rebuild `public/build/admin`. The
recommended path is the bundled command:

```bash
php bin/adminconsole sulu:admin:update-build
```

It downloads a prebuilt bundle when possible and falls back to a local build
(requires Node) if you have custom JS. Manual builds run from `assets/admin`
(`npm install && npm run build`); bun is supported experimentally. If installs
fail, clear all `node_modules` and lockfiles under the project and the
`vendor/sulu` JS trees, optionally `npm cache clean --force`.

**Website assets are separate.** When using Webpack Encore for the website,
keep the admin build untouched: move Flex's generated `assets/*` into
`assets/website/`, and point Encore at `public/build/website` (output path,
public path, `assets.yaml` manifest, `webpack_encore.yaml`). If a website build
wiped admin assets, restore with `git checkout public/build/admin` or rerun
`sulu:admin:update-build`. Encore details:
[references/admin-frontend-build.md](references/admin-frontend-build.md).
