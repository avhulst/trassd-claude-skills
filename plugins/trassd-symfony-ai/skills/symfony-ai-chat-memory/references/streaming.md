# Streaming chat responses

`ChatInterface::stream()` returns a `\Generator` that yields `DeltaInterface`
deltas as the model produces them. Filter for `TextDelta` to print incremental
text:

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Platform\Bridge\OpenAi\Factory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Result\Stream\Delta\TextDelta;

$platform = Factory::createPlatform($apiKey);
$agent = new Agent($platform, 'gpt-4o-mini');
$chat = new Chat($agent, new InMemoryStore());

$chat->initiate(new MessageBag(
    Message::forSystem('You are a helpful assistant.'),
));

foreach ($chat->stream(Message::ofUser('Tell me a story about the sun')) as $delta) {
    if ($delta instanceof TextDelta) {
        echo $delta;
    }
}
```

## Persistence

Once the stream is **fully consumed**, the assistant message is automatically
persisted to the message store alongside the user message — so history stays
up to date with no extra code. Always drain the generator (e.g. loop to the
end) or the assistant turn will not be saved.

## Caveat

Streaming with the Session-backed message store
(`Symfony\AI\Chat\Bridge\Session\MessageStore`) is **not recommended** because
of implementation limitations. Use a different store (InMemory, Cache, Redis,
Doctrine DBAL, …) when you need streaming.
