# Extension patterns

## Controller configuration in route defaults (2022-02-09)

**Decision:** Controller-level behaviour that used to be set with custom
annotations (`@LoginRequired`, `@Acl`, `@RouteScope`, `@ContextTokenRequired`,
`@Captcha`) is now configured in the `defaults` of the Symfony `#[Route]`
attribute (`_loginRequired`, `_acl`, `_routeScope`, `_contextTokenRequired`,
`_captcha`). Custom annotations were bound to the implementing class, forcing
every decorator to copy them; route defaults travel with the route definition.

**Rule for extensions:** Configure your controllers via `#[Route(...,
defaults: [...])]`, not custom annotations. To run code around a controller,
decorate it (if it has an abstract base) or hook
`KernelEvents::REQUEST`/`RESPONSE`.

```php
#[Route(
    path: '/store-api/product',
    name: 'store-api.product.search',
    methods: ['GET', 'POST'],
    defaults: ['_loginRequired' => true]
)]
public function search(): Response { /* ... */ }
```

## Mocking repositories (2023-04-01)

**Decision:** Testing a class that depends on an `EntityRepository` used to
require hand-building `EntitySearchResult`/`IdSearchResult` objects. The core
provides `Shopware\Tests\Unit\Common\Stubs\DataAbstractionLayer\StaticEntityRepository`,
whose constructor takes an ordered array of results (`EntityCollection`,
`EntitySearchResult`, `AggregationResultCollection`, or an id array for
`searchIds`) returned in sequence by successive `search`/`searchIds` calls.

**Rule for extensions:** Inject a `StaticEntityRepository` seeded with the
results your unit under test expects, instead of mocking `search` with manually
constructed result objects.

```php
$repository = new StaticEntityRepository([
    new UnitCollection([new UnitEntity(), new UnitEntity()]),
]);
$service = new SomeClass($repository);
```

## Choosing between decoration and events

Two ADRs above (in [core-and-dal.md](core-and-dal.md)) govern *how* you extend
core behaviour: the decoration pattern (2020-11-25) and the event-based
extension system (2024-06-18). Prefer subscribing to an `Extension` event point
when the core flow exposes one; fall back to decorating the abstract service
otherwise.
</content>
