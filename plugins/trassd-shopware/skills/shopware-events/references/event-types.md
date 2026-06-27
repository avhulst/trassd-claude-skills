# Common event types and how to use them

Companion to `SKILL.md`. All patterns are doc-grounded; class names verified
against `Shopware\Core\Framework\Event` and `…\DataAbstractionLayer\Event`.

## DAL write event with a version guard

Some entities (orders, products) are versioned, so a write event may be
dispatched multiple times for the same entity across versions. React only to
the live version:

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Subscriber;

use Shopware\Core\Content\Product\ProductEvents;
use Shopware\Core\Defaults;
use Shopware\Core\Framework\DataAbstractionLayer\Event\EntityWrittenEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class MySubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            ProductEvents::PRODUCT_WRITTEN_EVENT => 'onProductWritten',
        ];
    }

    public function onProductWritten(EntityWrittenEvent $event): void
    {
        if ($event->getContext()->getVersionId() !== Defaults::LIVE_VERSION) {
            return;
        }
        // ...
    }
}
```

DAL events follow `entity_name.event` (`product.written`, `product.deleted`,
`custom_entity.written`). Core entities expose constant classes such as
`Shopware\Core\Content\Product\ProductEvents`; for entities without one, use the
raw string:

```php
public static function getSubscribedEvents(): array
{
    return [
        ProductEvents::PRODUCT_LOADED_EVENT => 'onProductsLoaded',
        'custom_entity.written' => 'onCustomEntityWritten',
    ];
}
```

## Criteria events — modify the query before load

Many routes dispatch a "criteria" event before loading entities via the DAL,
letting you add/remove filters and associations. Example: the product listing
route dispatches `ProductListingCriteriaEvent` before delegating:

```php
$this->eventDispatcher->dispatch(
    new ProductListingCriteriaEvent($request, $criteria, $context)
);
```

Find these by searching `CriteriaEvent`. They are not auto-generated, so they
do not exist for every entity.

## Route events

Fine-grained aliases of Symfony kernel events, thrown per route. Replace
`{route}` with the Symfony route name (e.g. `store-api.product.listing`).

| Event name        | Scope      | Event type                                            |
|-------------------|------------|-------------------------------------------------------|
| `{route}.request` | Global     | `Symfony\Component\HttpKernel\Event\RequestEvent`     |
| `{route}.response`| Global     | `Symfony\Component\HttpKernel\Event\ResponseEvent`    |
| `{route}.render`  | Storefront | `Shopware\Storefront\Event\StorefrontRenderEvent`     |
| `{route}.encode`  | Store-API  | `Symfony\Component\HttpKernel\Event\ResponseEvent`    |
| `{route}.controller` | Global  | `Symfony\Component\HttpKernel\Event\ControllerEvent`  |

```php
public static function getSubscribedEvents(): array
{
    return [
        'store-api.product.listing.request' => 'onListingRequest',
        'store-api.product.listing.encode'  => 'onListingEncode',
    ];
}
```

## Page-loaded events

When a storefront page is rendered a "page is being loaded" event fires (e.g.
`GenericPageLoadedEvent`, dispatched by the generic page loader). Subscribe to
add meta information to the page. Find them by searching `PageLoadedEvent`.

## Business events

Business events fire on commerce actions (e.g. a customer registered, an order
was placed). There is frequently a `Before…` variant fired *before* the action,
which you can use to intercept or modify it (for example a customer login event
and its before-login counterpart). Each event exposes different data, so inspect
the specific event class.

## Manually-fired ("general PHP") events

You may find code firing an event directly:

```php
$someEvent = new SomeEvent($parameters, $moreParameters);
$this->eventDispatcher->dispatch($someEvent, $someEvent->getName());
```

The second `dispatch()` argument is the event name; if omitted the class name is
the fallback. Subscribe by name or by class:

```php
public static function getSubscribedEvents(): array
{
    return [
        'some_event'      => 'registeringToSomeEvent',
        SomeEvent::class   => 'registeringToSomeEvent', // when no name is applied
    ];
}
```
