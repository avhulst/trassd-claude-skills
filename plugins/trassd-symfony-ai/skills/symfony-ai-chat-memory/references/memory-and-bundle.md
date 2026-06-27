# Agent memory

Memory gives an agent stable knowledge of the user without cluttering the
conversation history: providers inject content into the **system prompt** via
the `Symfony\AI\Agent\Memory\MemoryInputProcessor`. Requires `symfony/ai-agent`.

## Static facts

`StaticMemoryProvider` holds a fixed list of always-available facts:

```php
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

$personalFacts = new StaticMemoryProvider(
    'My name is Wilhelm Tell',
    'I wish to be a swiss national hero',
    'I am struggling with hitting apples but want to be professional with the bow and arrow',
);
```

## Wiring the processor

`MemoryInputProcessor` loads the providers' facts and prepends them to the
system prompt on each call. It runs alongside other processors such as
`SystemPromptInputProcessor`; processors are applied in order:

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;

$memoryProcessor = new MemoryInputProcessor([$personalFacts]);

$agent = new Agent($platform, 'gpt-4o-mini', [$systemPromptProcessor, $memoryProcessor]);
```

Because the providers persist across calls, the conversation stays personalized
over time.

## Semantic recall

For a large knowledge base, swap the static provider for `EmbeddingProvider`,
which retrieves relevant context by semantic similarity. Pass it to
`MemoryInputProcessor` the same way:

```php
use Symfony\AI\Agent\Memory\EmbeddingProvider;

$embeddingsMemory = new EmbeddingProvider($platform, $model, $store);
```

## AI Bundle

Configure memory declaratively on an agent:

```yaml
# config/packages/ai.yaml
ai:
    agent:
        trainer:
            model: 'gpt-4o-mini'
            prompt:
                text: 'Provide short, motivating claims'
            memory: 'You are a professional trainer with personalized advice'
```

For a dynamic provider, point `memory` at a service instead.

## Best practices

- **Keep static memory concise** — only essential facts.
- **Separate concerns** — system prompt for behavior, memory for context.
- **Mind token usage** — memory content consumes input tokens.
