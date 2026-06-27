# Dynamic Toolbox (runtime tools)

Companion to `SKILL.md`. A dynamic toolbox decorates an existing
`ToolboxInterface`, letting you add, remove, rename, or re-describe tools at
runtime.

## The decorator skeleton

Implement `ToolboxInterface` (`getTools()` + `execute()`) and delegate to the
inner toolbox. In the AI Bundle, `#[AsDecorator('ai.toolbox.<agent>')]` targets a
specific agent's toolbox and `#[AutowireDecorated]` injects the original:

```php
namespace App;

use Symfony\AI\Agent\Toolbox\ToolboxInterface;
use Symfony\AI\Agent\Toolbox\ToolResult;
use Symfony\AI\Platform\Result\ToolCall;
use Symfony\Component\DependencyInjection\Attribute\AsDecorator;
use Symfony\Component\DependencyInjection\Attribute\AutowireDecorated;

#[AsDecorator('ai.toolbox.blog')]
class DynamicToolbox implements ToolboxInterface
{
    public function __construct(
        #[AutowireDecorated] private ToolboxInterface $innerToolbox,
    ) {
    }

    public function getTools(): array
    {
        return $this->innerToolbox->getTools();
    }

    public function execute(ToolCall $toolCall): ToolResult
    {
        return $this->innerToolbox->execute($toolCall);
    }
}
```

`getTools()` returns `Symfony\AI\Platform\Tool\Tool` objects; mutate that list to
change what the LLM sees.

## Re-describe a tool

Replace the matching `Tool` with a new one, reusing its reference and parameters:

```php
use Symfony\AI\Platform\Tool\Tool;

public function getTools(): array
{
    $tools = $this->innerToolbox->getTools();
    foreach ($tools as $i => $tool) {
        if ('similarity_search' !== $tool->getName()) {
            continue;
        }
        $tools[$i] = new Tool(
            $tool->getReference(),
            $tool->getName(),
            'Similarity search, but always add the word "please" to the searchTerm.',
            $tool->getParameters(),
        );
    }

    return $tools;
}
```

This is handy to experiment with descriptions or to trim tokens for complex
tools.

## Remove a tool (feature toggle)

```php
public function getTools(): array
{
    $tools = $this->innerToolbox->getTools();

    if (false === $this->clockEnabled) {
        $tools = array_filter($tools, static fn (Tool $t) => 'clock' !== $t->getName());
    }

    return $tools;
}
```

## Add a tool and handle its execution

Append a `Tool` with an `ExecutionReference` and a JSON-Schema parameter array,
then intercept its name in `execute()`:

```php
use Symfony\AI\Platform\Tool\ExecutionReference;
use Symfony\AI\Platform\Tool\Tool;

public function getTools(): array
{
    $tools = $this->innerToolbox->getTools();

    $tools[] = new Tool(
        new ExecutionReference(self::class), // required; not used for this tool
        'echo',
        'Echoes the input provided to it.',
        [
            'type' => 'object',
            'properties' => [
                'input' => ['type' => 'string', 'description' => 'text to echo'],
            ],
            'required' => ['input'],
            'additionalProperties' => false,
        ],
    );

    return $tools;
}

public function execute(ToolCall $toolCall): ToolResult
{
    if ('echo' === $toolCall->getName()) {
        $args = $toolCall->getArguments();

        return new ToolResult($toolCall, strtoupper($args['input']));
    }

    return $this->innerToolbox->execute($toolCall);
}
```

The new tool is now available to the agent alongside the existing ones.
