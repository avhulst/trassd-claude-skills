y---
name: shopware-events
description: >-
  React to and define events in Shopware 6 — finding events, writing event
  subscribers/listeners, and dispatching custom events. Use when adding an event
  subscriber to a plugin, listening to a business or DAL event (e.g.
  product.loaded, product.written, order placed), defining and firing a custom
  event class, or working out which event to hook into.
---

# Shopware Events

Shopware is built on the Symfony event dispatcher. You extend the platform by
**listening** to events fired at specific actions (entity reads/writes, page
loads, business actions) and by **dispatching** your own events that other
extensions can react to.

## Listening with an event subscriber

The idiomatic way to listen is a subscriber class implementing
`Symfony\Component\EventDispatcher\EventSubscriberInterface`. It works exactly
like a Symfony subscriber.

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Subscriber;

use Shopware\Core\Content\Product\ProductEvents;
use Shopware\Core\Framework\DataAbstractionLayer\Event\EntityLoadedEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class MySubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        // <event to listen to> => <method to execute>
        return [
            ProductEvents::PRODUCT_LOADED_EVENT => 'onProductsLoaded',
        ];
    }

    public function onProductsLoaded(EntityLoadedEvent $event): void
    {
        // $event->getEntities()
    }
}
```

Rules:

- Place the class under `<plugin root>/src/Subscriber` (convention, not required).
- The subscribed method receives the **event instance** for that event; check
  the event class to learn what data it exposes.
- `getSubscribedEvents()` keys may be **event-class constants** (e.g.
  `ProductEvents::PRODUCT_LOADED_EVENT`), the **class name** (`SomeEvent::class`),
  or the **string name** (`'custom_entity.written'`).

## Registering the subscriber

The subscriber must be a service tagged `kernel.event_subscriber`, loaded from
your plugin's `src/Resources/config/services.php`:

```php
<?php declare(strict_types=1);

use Swag\BasicExample\Subscriber\MySubscriber;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (ContainerConfigurator $configurator): void {
    $services = $configurator->services();

    $services->set(MySubscriber::class)
        ->tag('kernel.event_subscriber');
};
```

That tag is the only difference from a plain service. (Symfony autoconfigure
adds the tag automatically for classes implementing `EventSubscriberInterface`,
but Shopware plugin service files commonly declare it explicitly.)

## DAL and other common events

- **DAL events** fire on every entity read/write/delete and follow
  `entity_name.event`, e.g. `product.written`, `product.deleted`,
  `custom_entity.written`. Core entities also expose **event constant classes**
  (e.g. `Shopware\Core\Content\Product\ProductEvents`) — find them by searching
  for `@Event`. Write listeners receive a
  `Shopware\Core\Framework\DataAbstractionLayer\Event\EntityWrittenEvent`;
  load listeners receive an `EntityLoadedEvent`.
- **Versioning:** some entities (orders, products) dispatch events for multiple
  versions. Guard against non-live versions when needed:
  ```php
  if ($event->getContext()->getVersionId() !== \Shopware\Core\Defaults::LIVE_VERSION) {
      return;
  }
  ```
- **Page-loaded events** fire when a storefront page is rendered (e.g.
  `GenericPageLoadedEvent`) — find them by searching `PageLoadedEvent`. Use them
  to add data to a page.
- **Criteria events** (search `CriteriaEvent`, e.g. `ProductListingCriteriaEvent`)
  let you modify the `Criteria` before an entity is loaded — add/remove filters
  or associations. Not guaranteed to exist for every entity.
- **Route events** are fine-grained aliases of Symfony kernel events, named
  `{route}.request`, `{route}.response`, `{route}.render`, etc., where `{route}`
  is the Symfony route name (e.g. `store-api.product.listing.request`).
- **Business events** fire on commerce actions (customer registered, order
  placed), often with a `Before…` variant you can intercept before the action.

For a fuller catalog with examples (versioning guard, criteria, route, business
events) see [references/event-types.md](references/event-types.md).

## Defining and firing a custom event

Implement one of the Shopware event interfaces so listeners get a `Context`
(and, for sales-channel events, a `SalesChannelContext`):

- `Shopware\Core\Framework\Event\ShopwareEvent` — base interface, provides
  `getContext(): Context`.
- `Shopware\Core\Framework\Event\ShopwareSalesChannelEvent` — extends
  `ShopwareEvent`, adds `getSalesChannelContext(): SalesChannelContext`.
- `Shopware\Core\Framework\Event\SalesChannelAware` — provides the sales channel id.
- `Shopware\Core\Framework\Event\GenericEvent` — for giving the event an explicit
  name (like DAL events) instead of identifying it by class.
- `Shopware\Core\Framework\Event\NestedEvent` — base class for events that wrap
  other events.

```php
class ExampleEvent implements \Shopware\Core\Framework\Event\ShopwareSalesChannelEvent
{
    public function __construct(
        protected ExampleEntity $exampleEntity,
        protected \Shopware\Core\System\SalesChannel\SalesChannelContext $salesChannelContext,
    ) {}

    public function getExample(): ExampleEntity { return $this->exampleEntity; }
    public function getContext(): \Shopware\Core\Framework\Context { return $this->salesChannelContext->getContext(); }
    public function getSalesChannelContext(): \Shopware\Core\System\SalesChannel\SalesChannelContext { return $this->salesChannelContext; }
}
```

Fire it via the injected `event_dispatcher`
(`Symfony\Contracts\EventDispatcher\EventDispatcherInterface`):

```php
public function fireEvent(ExampleEntity $entity, SalesChannelContext $context): void
{
    $this->eventDispatcher->dispatch(new ExampleEvent($entity, $context));
}
```

The optional second `dispatch()` argument is the event **name**; omit it and the
class name is used as the name. Subscribers then key on `ExampleEvent::class`
(or the explicit name). See
[references/custom-event.md](references/custom-event.md) for the full event class
and service wiring.

## Discovering which event to use

- **`debug:event-dispatcher`** — `bin/console debug:event-dispatcher` lists all
  registered listeners; pass an event name to inspect its listeners.
- **Symfony profiler** — the "Events" tab shows every event fired in the current
  request, with the exact name to subscribe to.
- **Search the code** — useful terms: `@Event` (DAL constant classes),
  `extends NestedEvent` / `implements ShopwareEvent` (event classes),
  `->dispatch` (where events fire), `PageLoadedEvent`, `CriteriaEvent`.
- **Inspect service definitions** — a service that injects `event_dispatcher` /
  `EventDispatcherInterface` likely fires events; read its code to find them.
