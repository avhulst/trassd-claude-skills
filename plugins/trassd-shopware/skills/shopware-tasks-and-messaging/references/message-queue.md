# Message queue — envelopes, priority, transport

Shopware integrates the Symfony Messenger component (and Enqueue) for sending
and handling asynchronous messages. A message is a serializable PHP object; the
bus wraps it in an envelope on dispatch.

## Message placement and interfaces

Messages typically live under `src/MessageQueue/Message/`. By default messages
are handled synchronously. To change behaviour:

- `Shopware\Core\Framework\MessageQueue\AsyncMessageInterface` — handle
  asynchronously.
- `Shopware\Core\Framework\MessageQueue\LowPriorityMessageInterface` —
  asynchronous on the `low_priority` queue (Shopware 6.5.7.0+; configuring the
  messenger to consume that queue fails if it does not exist).

## Dispatching with metadata (stamps)

To attach metadata, dispatch a `Symfony\Component\Messenger\Envelope` with
stamps instead of the bare message. Example with a delay:

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Stamp\DelayStamp;

public function send(string $text): void
{
    $message = new SmsNotification($text);
    $this->bus->dispatch(
        (new Envelope($message))->with(new DelayStamp(5000)) // ms
    );
}
```

The bus to inject is `Symfony\Component\Messenger\MessageBusInterface`
(service `messenger.default_bus`).

## Routing / transport overrides

Route specific messages to the `low_priority` queue via `shopware.yaml` rather
than (or in addition to) implementing the interface:

```yaml
# config/packages/shopware.yaml
shopware:
    messenger:
        routing_overwrite:
            'Your\Custom\Message': low_priority
```

The same mechanism can override the transport for a message that implements
`LowPriorityMessageInterface` back to `async`:

```yaml
shopware:
    messenger:
        routing_overwrite:
            'Shopware\Core\Framework\MessageQueue\LowPriorityMessageInterface': low_priority
            'Your\Custom\LowPriorityMessage': async
```

## Handlers

A handler is invoked once the message is dispatched by the `handle_messages`
middleware. Mark the class `#[AsMessageHandler]`, implement `__invoke()` typed to
the message, and register it with the `messenger.message_handler` tag. Multiple
handlers may handle the same message.

## Middleware

The default bus runs messages through middleware (e.g. `handle_messages`). Custom
middleware can be added to the bus; for the routing-by-priority behaviour above,
core ships a `RoutingOverwriteMiddleware`.
