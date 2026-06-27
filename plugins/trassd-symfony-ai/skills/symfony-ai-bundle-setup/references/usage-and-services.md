# Referencing configured agents and stores as services

Every section of `ai.yaml` registers container services with predictable ids:

| Config section | Service id pattern |
| --- | --- |
| `platform.<name>` | `ai.platform.<name>` |
| `agent.<name>` | `ai.agent.<name>` |
| `store.<type>.<name>` | `ai.store.<type>.<name>` |
| `vectorizer.<name>` | `ai.vectorizer.<name>` |
| `indexer.<name>` | `ai.indexer.<name>` |
| `retriever.<name>` | `ai.retriever.<name>` |
| `message_store.<type>.<name>` | `ai.message_store.<type>.<name>` |
| `chat.<name>` | `ai.chat.<name>` |
| `multi_agent.<name>` | `ai.multi_agent.<name>` |

## Injecting a single agent

With exactly one agent configured, autowire `AgentInterface` directly:

```php
use Symfony\AI\Agent\AgentInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

final readonly class AssistantService
{
    public function __construct(
        private AgentInterface $agent,
    ) {
    }

    public function ask(string $question): string
    {
        $messages = new MessageBag(Message::ofUser($question));

        return $this->agent->call($messages)->getContent();
    }
}
```

## Injecting a specific agent

When several agents exist, target one by service id:

```php
use Symfony\AI\Agent\AgentInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

public function __construct(
    #[Autowire(service: 'ai.agent.researcher')]
    private AgentInterface $researcher,
) {
}
```

The same `#[Autowire(service: '...')]` pattern injects a specific multi-agent
orchestrator (`ai.multi_agent.<name>`) or retriever (`ai.retriever.<name>`).

## Store injection by alias

For each configured store the bundle registers two aliases:

- **Simple alias** `StoreInterface $<name>` — references the store by name (the
  first occurrence wins when the name is shared across types).
- **Type-prefixed alias** `StoreInterface $<type><Name>` (camelCase) — explicit
  disambiguation.

Given:

```yaml
ai:
    store:
        memory:
            main:
                strategy: 'cosine'
            products:
                strategy: 'manhattan'
        chromadb:
            main:
                collection: 'documents'
```

the following are autowirable: `$main` (memory `main`, first occurrence),
`$memoryMain`, `$chromadbMain`, `$products`, `$memoryProducts`. Prefer the
type-prefixed aliases when a name is shared:

```php
use Symfony\AI\Store\StoreInterface;

final readonly class DocumentService
{
    public function __construct(
        private StoreInterface $main,           // memory main (first occurrence)
        private StoreInterface $chromadbMain,   // chromadb main
        private StoreInterface $memoryProducts, // memory products
    ) {
    }
}
```

## Custom tools

Annotate a service with `#[AsTool]` (it is tagged `ai.tool` and auto-registered),
then opt the agent in:

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool('company_name', 'Provides the name of your company')]
final class CompanyName
{
    public function __invoke(): string
    {
        return 'ACME Corp.';
    }
}
```

Restrict a tool with `#[IsGrantedTool('ROLE_ADMIN')]`
(`Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool`, requires
`symfony/security-core`). Multiple attributes combine with logical AND.
