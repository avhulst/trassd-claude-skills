# Defining and firing a custom event

Companion to `SKILL.md`. Interface names verified against
`Shopware\Core\Framework\Event` (`ShopwareEvent`, `ShopwareSalesChannelEvent`,
`SalesChannelAware`, `GenericEvent`, `NestedEvent`).

## Event interfaces and classes

- `ShopwareEvent` — base interface, provides `getContext(): Context` needed by
  almost all events.
- `ShopwareSalesChannelEvent` — extends `ShopwareEvent`, additionally provides
  `getSalesChannelContext(): SalesChannelContext`.
- `SalesChannelAware` — provides the sales channel id.
- `GenericEvent` — use when the event should carry an explicit string name
  (like the DAL `product.written` events) rather than being identified by class.
- `NestedEvent` — base class for events that wrap other events (for example an
  entity-deleted event that extends an entity-written event).

## The event class

```php
// <plugin root>/src/Core/Content/Example/Event/ExampleEvent.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Core\Content\Example\Event;

use Shopware\Core\Framework\Context;
use Shopware\Core\Framework\Event\ShopwareSalesChannelEvent;
use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Swag\BasicExample\Core\Content\Example\ExampleEntity;

class ExampleEvent implements ShopwareSalesChannelEvent
{
    protected ExampleEntity $exampleEntity;

    protected SalesChannelContext $salesChannelContext;

    public function __construct(ExampleEntity $exampleEntity, SalesChannelContext $context)
    {
        $this->exampleEntity = $exampleEntity;
        $this->salesChannelContext = $context;
    }

    public function getExample(): ExampleEntity
    {
        return $this->exampleEntity;
    }

    public function getContext(): Context
    {
        return $this->salesChannelContext->getContext();
    }

    public function getSalesChannelContext(): SalesChannelContext
    {
        return $this->salesChannelContext;
    }
}
```

## Firing the event

Inject the `event_dispatcher` service
(`Symfony\Contracts\EventDispatcher\EventDispatcherInterface`) and call
`dispatch()`:

```php
// <plugin root>/src/Service/ExampleEventService.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Service;

use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Swag\BasicExample\Core\Content\Example\Event\ExampleEvent;
use Swag\BasicExample\Core\Content\Example\ExampleEntity;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

class ExampleEventService
{
    private EventDispatcherInterface $eventDispatcher;

    public function __construct(EventDispatcherInterface $eventDispatcher)
    {
        $this->eventDispatcher = $eventDispatcher;
    }

    public function fireEvent(ExampleEntity $exampleEntity, SalesChannelContext $context): void
    {
        $this->eventDispatcher->dispatch(new ExampleEvent($exampleEntity, $context));
    }
}
```

The optional second `dispatch()` argument is the event name; omit it to fall
back to the class name. Other extensions then listen with a subscriber keyed on
`ExampleEvent::class` (or the explicit name) — see `SKILL.md` for the subscriber
side.
