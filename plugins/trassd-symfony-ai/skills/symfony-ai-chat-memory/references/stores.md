# Message stores

The Chat component's main extension point is
`Symfony\AI\Chat\MessageStoreInterface`. The `Chat` loads history before a turn
and saves it after, so the store you pick decides the conversation's scope and
durability — the rest of your code is unchanged.

## Built-in stores

Choose by lifetime:

| Store | Scope / lifetime |
| --- | --- |
| `Symfony\AI\Chat\InMemory\Store` | current process / request only |
| Session bridge (`Bridge\Session\MessageStore`) | current user session |
| Cache, Redis, Pogocache, Cloudflare | external, typically short-lived |
| Doctrine DBAL (`Bridge\Doctrine\DoctrineDbalMessageStore`), MongoDB, Meilisearch, SurrealDb | long-term, durable |

## Custom store

Implement the two `MessageStoreInterface` methods:

```php
use Symfony\AI\Chat\MessageStoreInterface;
use Symfony\AI\Platform\Message\MessageBag;

class MyCustomStore implements MessageStoreInterface
{
    public function save(MessageBag $messages): void
    {
        // persist the message bag
    }

    public function load(): MessageBag
    {
        // return the stored message bag
    }
}
```

## Managed stores (setup / drop)

Stores that need backing tables or indexes also implement
`Symfony\AI\Chat\ManagedStoreInterface`:

```php
use Symfony\AI\Chat\ManagedStoreInterface;
use Symfony\AI\Chat\MessageStoreInterface;

class MyCustomStore implements ManagedStoreInterface, MessageStoreInterface
{
    // save() / load() ...

    public function setup(array $options = []): void
    {
        // create tables / indexes
    }

    public function drop(): void
    {
        // drop the store and its messages
    }
}
```

## Console commands (AiBundle)

With the AiBundle, configure and manage a named store:

```yaml
# config/packages/ai.yaml
ai:
    message_store:
        cache:
            symfonycon:
                service: 'cache.app'
```

```terminal
php bin/console ai:message-store:setup symfonycon
php bin/console ai:message-store:drop symfonycon
```
