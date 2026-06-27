---
name: shopware-store-api-routes
description: >-
  Add and override Store-API routes in a Shopware 6 plugin — the AbstractRoute
  pattern, the #[Route] attribute with the store-api route scope, accepting a
  SalesChannelContext and Criteria, returning a *RouteResponse struct that
  extends StoreApiResponse, registering the controller as a service, and
  decorating existing core routes. Use when adding a new Store-API endpoint
  (e.g. /store-api/...), wrapping a route for the Storefront, or overriding /
  extending a core Store-API route.
---

# Shopware Store-API routes

Store-API routes are **service-based controllers** that expose focused,
cacheable, decoratable endpoints under `/store-api/...`. They are the only
place business logic for the storefront should live — Storefront controllers
and page loaders call routes, never repositories directly.

## Core rules

- **Do not** implement the deprecated Sales Channel API (deprecated since 6.4).
- Define every Store-API controller **as a service** and use **named routes**.
- Each route class (or each API method) carries the route scope attribute:
  `#[Route(defaults: [PlatformRequest::ATTRIBUTE_ROUTE_SCOPE => [StoreApiRouteScope::ID]])]`.
  `StoreApiRouteScope::ID` is `'store-api'`.
- A route represents **a single, focused functionality**.
- A `load`/handler method accepts a `SalesChannelContext` (and usually a
  `Criteria` for search endpoints) and must **return a `StoreApiResponse`**, so
  it can be converted to JSON.
- A route response **may contain only one object** (the `$object` property).
- Routes are built on the **abstract route + concrete route** pattern so they
  can be decorated. Response classes that decorate/extend a response must
  extend `StoreApiResponse`.

## The AbstractRoute pattern (why)

Decoration in Shopware requires an abstract base. Every route is split in two:

1. An **abstract class** declaring `getDecorated()` and the handler method
   (e.g. `load(...)`) with its return type fixed to a concrete `*RouteResponse`.
2. A **concrete class** extending it, carrying the `#[Route]` attributes and
   doing the work. Its `getDecorated()` throws `DecorationPatternException`
   (it has no decorator yet).

A decorator later extends the **same abstract class**, takes the inner route in
its constructor, and returns it from `getDecorated()`. This is what makes
core and custom routes overridable. See
[references/add-route.md](references/add-route.md).

```php
abstract class AbstractExampleRoute
{
    abstract public function getDecorated(): AbstractExampleRoute;

    abstract public function load(Criteria $criteria, SalesChannelContext $context): ExampleRouteResponse;
}
```

```php
#[Route(defaults: [PlatformRequest::ATTRIBUTE_ROUTE_SCOPE => [StoreApiRouteScope::ID]])]
class ExampleRoute extends AbstractExampleRoute
{
    public function __construct(private readonly EntityRepository $exampleRepository) {}

    public function getDecorated(): AbstractExampleRoute
    {
        throw new DecorationPatternException(self::class);
    }

    #[Route(path: '/store-api/example', name: 'store-api.example.search', methods: ['GET', 'POST'], defaults: ['_entity' => 'swag_example'])]
    public function load(Criteria $criteria, SalesChannelContext $context): ExampleRouteResponse
    {
        return new ExampleRouteResponse($this->exampleRepository->search($criteria, $context->getContext()));
    }
}
```

Notes on the attribute defaults:
- `_routeScope` (`PlatformRequest::ATTRIBUTE_ROUTE_SCOPE`) is set to
  `[StoreApiRouteScope::ID]` — without it the route is not recognized as a
  Store-API route.
- `_entity` marks the entity the endpoint returns; with it Shopware can resolve
  the incoming `Criteria` from the request body automatically.
- The `name` is the internal named route (`store-api.<area>.<action>`).

## The response struct

Create a `*RouteResponse` that extends
`Shopware\Core\System\SalesChannel\StoreApiResponse`. The parent holds the
single `$object` (set via the constructor) and serializes it to JSON. Add typed
getters for ergonomic access.

```php
/** @property EntitySearchResult<ExampleCollection> $object */
class ExampleRouteResponse extends StoreApiResponse
{
    public function getExamples(): ExampleCollection
    {
        return $this->object->getEntities();
    }
}
```

## Registering the route

Register the controller as a service and import the route classes for
attribute routing. Full snippets in
[references/add-route.md](references/add-route.md).

- **services** (`src/Resources/config/services.php`): `->set(ExampleRoute::class)`
  with its dependencies (e.g. `service('swag_example.repository')`).
- **routes** (`src/Resources/config/routes.php`): import the route classes,
  e.g. `$routes->import('../../Core/**/*Route.php', 'attribute');`.

Verify with `./bin/console debug:router store-api.example.search`. Optionally
document the endpoint by dropping an OpenAPI JSON in
`src/Resources/Schema/StoreApi/`.

## Overriding / decorating a core route

To extend an existing Store-API route, **decorate** it — never copy its logic:

1. Create a class extending the route's **abstract** class
   (e.g. `AbstractExampleRoute`).
2. Constructor accepts the inner `AbstractExampleRoute`; `getDecorated()`
   returns it.
3. In the handler, call `$this->getDecorated()->load(...)` (or the inner
   instance), then add your data / headers, and return the (still
   `StoreApiResponse`) result.
4. Register the decorator **after** the original with `->decorate(Original::class)`
   and pass `service('.inner')` as the decorated argument.

```php
#[Route(defaults: [PlatformRequest::ATTRIBUTE_ROUTE_SCOPE => [StoreApiRouteScope::ID]])]
class ExampleRouteDecorator extends AbstractExampleRoute
{
    public function __construct(
        private readonly EntityRepository $exampleRepository,
        private readonly AbstractExampleRoute $decorated,
    ) {}

    public function getDecorated(): AbstractExampleRoute
    {
        return $this->decorated;
    }

    #[Route(path: '/store-api/example', name: 'store-api.example.search', methods: ['GET', 'POST'])]
    public function load(Criteria $criteria, SalesChannelContext $context): ExampleRouteResponse
    {
        $response = $this->decorated->load($criteria, $context);
        // augment $response here, e.g. headers or extra associations
        return $response;
    }
}
```

Service registration for the decorator is in
[references/override-route.md](references/override-route.md).

## Backwards-compatible / versioned responses

The Store-API contract is consumed by external clients, so responses must stay
**backwards-compatible**:

- A route response carries **one object only** — keep that shape stable.
- **Add**, don't remove or rename, fields in responses. Removing or changing a
  field's meaning is a breaking change.
- Only fields flagged `ApiAware` in the entity definition are exposed through
  the API — gate new fields deliberately.
- Decorate to **extend** behaviour (add data, headers); preserve the existing
  contract for callers that don't know about your addition.

## Exposing a route to the Storefront

If the storefront JS needs the data, add a thin Storefront controller in the
`storefront` route scope (`StorefrontRouteScope::ID`) that injects the
**abstract** route and delegates to it (add `'XmlHttpRequest' => 'true'` to the
route defaults for AJAX). The controller does no business logic and never
touches a repository. Example in
[references/add-route.md](references/add-route.md).
