# Core architecture & the DAL

## Decoration pattern (2020-11-25)

**Decision:** Shopware no longer defines services as interfaces; decoratable
services are abstract classes. Abstract classes let the core add optional
parameters or whole new methods without breaking existing decorators, and the
mandatory `getDecorated()` method provides a fallback so a decorator that
doesn't yet implement a new method is simply skipped in the chain.

**Rule for extensions:** When you decorate a core service, extend its abstract
base class, type-hint the abstract class (not a concrete class), and implement
`getDecorated()` to return the next service in the chain. Call new methods
unconditionally — the fallback handles older links.

```php
class MyCustomerRoute extends AbstractCustomerRoute
{
    public function __construct(private AbstractCustomerRoute $decorated) {}

    public function getDecorated(): AbstractCustomerRoute
    {
        return $this->decorated;
    }

    public function load(Request $request, SalesChannelContext $context): CustomerResponse
    {
        // your logic, then delegate
        return $this->getDecorated()->load($request, $context);
    }
}
```

## When to use plain SQL or the DAL (2021-05-14)

**Decision:** Use the DAL wherever third parties must be able to extend the
loaded/returned data or where data integrity matters — Store-API, storefront
page loaders/controllers, admin API, and **all writes** (the DAL runs
validation, events and indexing). Use plain SQL only in entity indexers and
internal core components (theme compiler, request transformer, …) that are not
extension points and only process data internally.

**Rule for extensions:** Read and write through the entity repository. Drop to
the DBAL connection only for internal, non-extensible bulk processing such as
your own indexer. Never write business data with raw SQL — you bypass events
and indexing.

## DAL join filter (2020-11-19)

**Decision:** The DAL forms join-groups per multi-filter layer. Filters on a
to-many association placed in the *same* multi-filter share one join (all
conditions must hold for the same related row); filters added separately create
separate joins (the conditions may match different related rows).

**Rule for extensions:** To require that several conditions match the *same*
related entity, wrap them in one `AndFilter`/multi-filter. Adding them as
separate `addFilter()` calls changes the meaning — and the number of rows
returned.

```php
// All conditions must hold for the SAME category row:
$criteria->addFilter(new AndFilter([
    new EqualsFilter('product.categories.name', 'test-category'),
    new EqualsFilter('product.categories.active', true),
]));
```

## Technical concept: custom entities (2021-09-14)

**Decision:** Apps declare entities in `config/custom_entity.xml`. Each entity
is registered with a `custom_entity_` (or `ce_`) prefix, gets a real MySQL
table, an `IdField(id)` primary key, and a required `TranslatedField(label)`.
Supported field types: scalars, JSON fields, and linking associations
(many-to-one, many-to-many). New fields must be nullable or have a default;
changing a field's type is forbidden; removing a field drops the column.

**Rule for extensions:** Always prefix entity names with your vendor token to
avoid collisions (`custom_entity_swag_blog`). Mark an entity `store_api_aware`
to expose it via the Store-API — otherwise it is stripped from responses.
Generic CRUD admin-API routes are auto-provided at `/api/custom-entity-{entity}`
(or `/api/ce-{entity}`).

## Deprecate `autoload: true` in DAL associations (2023-02-02)

**Decision:** `autoload: true` on `OneToOne`/`ManyToOne` association fields
loads the association on every query whether or not it is used — a performance
cost (extra joins, hydration, larger payloads). It is deprecated in core.

**Rule for extensions:** Define associations with `autoload: false` and request
them explicitly when you need them via `$criteria->addAssociation(...)`. Don't
depend on a core association being autoloaded; it may be removed.

## Switch to UUIDv7 (2023-05-22)

**Decision:** Shopware switched the `Uuid` class from v4 to v7. v7's time-based
prefix keeps B-tree indexes compact, improving bulk-insert performance. v4 and
v7 are the same length and indistinguishable to Shopware, so the switch is safe.

**Rule for extensions:** Generate primary keys with `Uuid::randomHex()` (the
core `Uuid` class) so you automatically get v7 ordering. Don't hand-roll random
hex IDs.

## Transition to an event-based extension system (2024-06-18)

**Decision:** To reduce the proliferation of interfaces/abstract classes and
simplify forward/backward compatibility, the core is introducing event-based
extension points (`Extension` classes dispatched with `.pre`/`.post` events)
alongside decoration. The event carries public, readable payload properties and
a writable `result`.

**Rule for extensions:** Where a core flow exposes an `Extension` point, prefer
subscribing to its event over decorating a service. Subscribe to `<name>.pre`
to replace core behaviour: set `$event->result` and call
`$event->stopPropagation()`; subscribe to `<name>.post` to adjust the result.

```php
public static function getSubscribedEvents(): array
{
    return ['listing-loader.resolve.pre' => 'replace'];
}

public function replace(ResolveListingExtension $event): void
{
    $event->result = $this->repository->search(/* ... */, $event->context->getContext());
    $event->stopPropagation();
}
```
</content>
