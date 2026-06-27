# Tool parameters, validation, sources & robustness

Companion to `SKILL.md`. All identifiers are from `symfony/ai-agent`.

## Parameter JSON Schema

The `Toolbox` builds a JSON Schema for each tool from the `#[AsTool]` attribute,
the method argument types, and `@param` docblock comments. LLMs honor the schema
when producing arguments.

### `#[Schema]` attribute (validation rules)

Add `Symfony\AI\Platform\Contract\JsonSchema\Attribute\Schema` to method
arguments to express constraints:

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\Schema;

#[AsTool('my_tool', 'Example tool with parameter requirements.')]
final class MyTool
{
    /**
     * @param string        $name       The name of an object
     * @param int           $number     The number of an object
     * @param array<string> $categories List of valid categories
     */
    public function __invoke(
        #[Schema(pattern: '/([a-z0-1]){5}/')]
        string $name,
        #[Schema(minimum: 0, maximum: 10)]
        int $number,
        #[Schema(enum: ['tech', 'business', 'science'])]
        array $categories,
    ): string {
        // ...
    }
}
```

These rules are only emitted into the schema for the LLM to respect; Symfony AI
does NOT validate them itself unless you add the listener below.

### Schema from a file

Reference an external schema with `ref:` (no other `#[Schema]` args allowed then):

```php
public function __invoke(
    #[Schema(ref: __DIR__.'/schema.json')]
    array $data,
): string { /* ... */ }
```

### Automatic enum validation

PHP backed enums are validated automatically — no `#[Schema(enum: ...)]` needed:

```php
enum ContentType: string { case ARTICLE = 'article'; case NEWS = 'news'; }

#[AsTool('content_search', 'Search content.')]
final class ContentSearchTool
{
    /** @param ContentType $type The content type to search for */
    public function __invoke(ContentType $type): array { /* ... */ }
}
```

### Symfony Validator constraints

With `symfony/validator` installed, constraints on a value object drive the
schema:

```php
use Symfony\Component\Validator\Constraints as Assert;

class Person
{
    #[Assert\Length(max: 255)] public string $name;
    #[Assert\Range(min: 18)]   public int $age;
}

#[AsTool('person_lookup', 'Look up a person.')]
final class MyTool
{
    public function __invoke(Person $person): string { /* ... */ }
}
```

### Enforcing validation before execution

To actually reject bad arguments before the tool runs, register the built-in
`ValidateToolCallArgumentsListener` on the dispatcher passed to the `Toolbox`:

```php
use Symfony\AI\Agent\Toolbox\Event\ToolCallArgumentsResolved;
use Symfony\AI\Agent\Toolbox\EventListener\ValidateToolCallArgumentsListener;

$eventDispatcher->addListener(ToolCallArgumentsResolved::class, new ValidateToolCallArgumentsListener());
```

It throws `InvalidToolCallArgumentsException` on failure. The AI Bundle registers
this automatically. For parameters that may be one of several types, use
Symfony Serializer's `DiscriminatorMap` to generate an `anyOf` schema.

## Third-party tools (no `#[AsTool]` access)

Register external classes explicitly via `MemoryToolFactory`:

```php
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\ToolFactory\MemoryToolFactory;
use Symfony\Component\Clock\Clock;

$factory = (new MemoryToolFactory())
    ->addTool(Clock::class, 'clock', 'Get the current date and time', 'now');
$toolbox = new Toolbox([new Clock()], $factory);
```

Combine attribute-based and explicit tools with `ChainFactory` (first factory
wins, so it can override existing config):

```php
use Symfony\AI\Agent\Toolbox\ToolFactory\ChainFactory;
use Symfony\AI\Agent\Toolbox\ToolFactory\ReflectionToolFactory;

$toolbox = new Toolbox([...], new ChainFactory($memoryFactory, new ReflectionToolFactory()));
```

## Fault tolerance

Decorate the toolbox with `FaultTolerantToolbox` to turn tool errors (wrong tool
names, runtime exceptions) into readable messages for the LLM instead of hard
failures:

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

$toolbox = new FaultTolerantToolbox($innerToolbox);
```

To surface a specific message to the LLM, throw an exception implementing
`Symfony\AI\Agent\Toolbox\Exception\ToolExecutionExceptionInterface` and return
the text from `getToolCallResult()`.

## Tool sources (RAG-style attribution)

Enable `includeSources: true` on the `AgentProcessor`, implement
`HasSourcesInterface` with `HasSourcesTrait` in the tool, call `$this->addSource(new Source(...))`,
and read them afterwards from the result metadata:

```php
foreach ($result->getMetadata()->get('sources', []) as $source) {
    echo $source->getName().' '.$source->getReference();
}
```

## Built-in & RAG tools

The component ships ready-made tools as separate Composer packages — Wikipedia,
YouTube, Brave, SerpApi, and `SimilaritySearch` (RAG over the Store component via
a `Retriever`). Add them to the `Toolbox` alongside your custom tools, e.g.
`new Wikipedia(HttpClient::create())`.
