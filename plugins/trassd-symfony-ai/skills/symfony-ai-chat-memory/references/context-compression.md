# Context compression for long conversations

As history grows, token cost rises and you risk exceeding the model's context
window. Compress with a custom `Symfony\AI\Agent\InputProcessorInterface` whose
`processInput(Input $input)` rewrites the `MessageBag` before it is sent to the
model. Requires `symfony/ai-agent`.

Two strategies follow. Both gate on a `threshold` so short conversations are
left alone, and both preserve the system message.

## Sliding window

Discard older turns, keep the most recent ones. Fast and free, but loses
context.

```php
namespace App\Agent\InputProcessor;

use Symfony\AI\Agent\Input;
use Symfony\AI\Agent\InputProcessorInterface;
use Symfony\AI\Platform\Message\MessageBag;

final class SlidingWindowInputProcessor implements InputProcessorInterface
{
    public function __construct(
        private int $maxMessages = 10,
        private int $threshold = 20,
    ) {
    }

    public function processInput(Input $input): void
    {
        $messages = $input->getMessageBag();
        $nonSystemMessages = $messages->withoutSystemMessage()->getMessages();

        if (\count($nonSystemMessages) <= $this->threshold) {
            return;
        }

        $systemMessage = $messages->getSystemMessage();
        $recentMessages = \array_slice($nonSystemMessages, -$this->maxMessages);

        $input->setMessageBag(null !== $systemMessage
            ? new MessageBag($systemMessage, ...$recentMessages)
            : new MessageBag(...$recentMessages),
        );
    }
}
```

Register it on the agent (processors apply in order, on every call):

```php
use Symfony\AI\Agent\Agent;

$agent = new Agent($platform, 'gpt-4o-mini', [new SlidingWindowInputProcessor()]);
```

## Summarization

When older messages matter, summarize them with a separate (cheap) LLM call and
fold the summary into the system message; keep the last few turns verbatim.

```php
namespace App\Agent\InputProcessor;

use Symfony\AI\Agent\Input;
use Symfony\AI\Agent\InputProcessorInterface;
use Symfony\AI\Platform\Message\AssistantMessage;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\PlatformInterface;

final class SummarizationInputProcessor implements InputProcessorInterface
{
    public function __construct(
        private PlatformInterface $platform,
        private string $model = 'gpt-4o-mini',
        private int $threshold = 20,
        private int $keepRecent = 6,
    ) {
    }

    public function processInput(Input $input): void
    {
        $messages = $input->getMessageBag();
        $nonSystemMessages = $messages->withoutSystemMessage()->getMessages();

        if (\count($nonSystemMessages) <= $this->threshold) {
            return;
        }

        $toSummarize = \array_slice($nonSystemMessages, 0, -$this->keepRecent);
        $toKeep = \array_slice($nonSystemMessages, -$this->keepRecent);

        $summary = $this->platform->invoke(
            $this->model,
            new MessageBag(Message::ofUser(
                'Summarize this conversation concisely, focusing on key decisions '
                .'and current task state: '.\PHP_EOL.$this->formatMessages($toSummarize),
            )),
        )->asText();

        $systemContent = '';
        $systemMessage = $messages->getSystemMessage();
        if (null !== $systemMessage) {
            $systemContent = $systemMessage->getContent().\PHP_EOL.\PHP_EOL;
        }
        $systemContent .= '# Previous Conversation Summary'.\PHP_EOL.\PHP_EOL.$summary;

        $input->setMessageBag(new MessageBag(
            Message::forSystem($systemContent),
            ...$toKeep,
        ));
    }

    private function formatMessages(array $messages): string
    {
        $lines = [];
        foreach ($messages as $message) {
            if ($message instanceof UserMessage) {
                $lines[] = 'User: '.$message->asText();
            }

            if ($message instanceof AssistantMessage) {
                $lines[] = 'Assistant: '.$message->asText();
            }
        }

        return \implode(\PHP_EOL, $lines);
    }
}
```

## AI Bundle wiring

The `#[AsInputProcessor]` attribute registers a processor for one agent — or
for all agents when `agent` is omitted:

```php
use Symfony\AI\Agent\Attribute\AsInputProcessor;

#[AsInputProcessor(agent: 'my_agent')]
final class SummarizationInputProcessor implements InputProcessorInterface
{
    // ...
}
```

Wire the platform dependency:

```yaml
# config/services.yaml
services:
    App\Agent\InputProcessor\SummarizationInputProcessor:
        $platform: '@ai.platform.openai'
        $model: 'gpt-4o-mini'
```

## Best practices

- **Pick the right strategy** — sliding window is fast and free but loses
  context; summarization preserves it at the cost of latency and an extra call.
- **Use a smaller model** for summarization (e.g. `gpt-4o-mini`).
- **Tune the threshold** — start at 20–30 messages.
- **Never drop the system message.**
- **Keep 4–8 recent messages uncompressed** for immediate context.
