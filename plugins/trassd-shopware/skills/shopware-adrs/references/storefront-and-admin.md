# Storefront & admin

## Nested line item templates (2022-03-17)

**Decision:** The many per-area line-item templates (checkout-item,
offcanvas-item, confirm-item, order line-item, and their `-children` variants)
were consolidated into a single recursive base template
`Resources/views/storefront/component/line-item/line-item.html.twig`. It renders
all shop areas (appearance toggled by config vars), includes itself for nested
line items, and delegates to per-type templates (product, container, discount,
custom types).

**Rule for extensions:** Extend (or add your custom line-item type to) the
`component/line-item/line-item.html.twig` base template instead of the
deprecated `checkout-item*`/`offcanvas-item*`/`confirm-item`/`finish-item`/
account-order line-item templates.

## Storefront coding standards (2021-08-10)

**Decision:** Storefront controllers follow strict conventions: routes use the
`#[Route]` attribute with a `frontend.`-prefixed `name`, explicit HTTP methods,
and the storefront route scope on the class; each route has a single purpose and
a return type hint. Page-rendering (GET) routes delegate data loading to a
**PageLoader** (which gathers data via Store-API routes and returns a
Page object; PageletLoaders handle page fragments). Controllers contain **no
business logic** and **never use a repository directly** — every storefront
action must also be available as a Store-API route. Cacheable pages set
`_httpCache: true`; write operations use `createActionResponse` and Symfony
flash bags for user feedback.

**Rule for extensions:** Extend
`Shopware\Storefront\Controller\StorefrontController`, inject dependencies via
the constructor (assigned to private properties, declared as public services),
name routes `frontend.*`, move data loading into a PageLoader, and back every
action with a Store-API route. Keep business logic out of the controller.

```php
#[Route(
    path: '/example/{id}',
    name: 'frontend.example.page',
    defaults: ['_httpCache' => true, '_loginRequired' => true],
    methods: ['GET']
)]
public function page(string $id, Request $request, SalesChannelContext $context): Response
{
    $page = $this->pageLoader->load($request, $context);
    return $this->renderStorefront('@MyPlugin/.../index.html.twig', ['page' => $page]);
}
```

## TypeScript support for storefront JS (2022-06-24)

**Decision:** The storefront's existing Babel chain gained TypeScript support
via `@babel/preset-typescript`. To preserve compatibility, no public `.js` file
is replaced by a `.ts` file without proper deprecation; `.ts`/`.tsx` and `.js`
interoperate freely.

**Rule for extensions:** Write new storefront JavaScript in TypeScript and
incrementally convert your own JS to `.ts`. Don't rename a publicly imported
`.js` file to `.ts` without a deprecation path — downstream plugins may import
it.

## Admin extension API standards (2021-12-07)

**Decision:** Extensions add admin views/components through the
Admin-Extension-API without patching core components. Custom views render in
iFrames at **locations** with unique location IDs (`sw.location.is('...')`).
Existing areas are extended at **positions** identified by position IDs
(`sw.ui.tabs('sw-product-detail').addTabItem(...)`). **Component sections**
(`sw.ui.componentSection('<positionId>').add({...})`) are generic injection
points where extensions place prebuilt components (cards, etc.), optionally
hosting a custom iFrame view via a `locationId`. A Vue Devtools plugin surfaces
the available position IDs.

**Rule for extensions:** Inject UI through `sw.ui.*` extension points and
component sections keyed by the documented position/location IDs; render custom
content in iFrame locations. Don't override or fork core admin components to add
UI.

```js
sw.ui.componentSection('sw-manufacturer-card-custom-fields__before').add({
    component: 'card',
    props: { title: 'My card', locationId: 'my-app-card' },
});
```

## Replace Vuex with Pinia (2024-06-17)

**Decision:** Admin state moves from Vuex (`Shopware.State`) to Pinia, exposed
under `Shopware.Store` (a singleton with `list`/`get`/`register`/`unregister`).
`Shopware.State` is deprecated in 6.7 (warnings) and removed in 6.8 along with
the `vuex` package. Pinia stores have no mutations (fold them into actions),
`state` is an arrow function returning an object, and actions/getters use `this`
instead of a `state` argument.

**Rule for extensions:** Register admin stores with `Shopware.Store.register(...)`
(not `Shopware.State.registerModule`), write them in TypeScript, and export a
state type/interface reused for the `state` definition. Migrate existing Vuex
modules by moving mutations into actions and switching to `this`-based access.

```typescript
Shopware.Store.register({
    id: 'example',
    state: () => ({ id: '' }),
    getters: { idStart: () => this.id.substring(0, 4) },
    actions: { setId(id: string) { this.id = id; } },
});
```
</content>
