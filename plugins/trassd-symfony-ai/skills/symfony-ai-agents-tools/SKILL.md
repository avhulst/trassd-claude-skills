---
name: symfony-ai-agents-tools
description: >-
  Build Symfony AI Agents with tool calling — defining an agent, registering
  tools, dynamic tools, human-in-the-loop approval, and multi-agent
  orchestration. Covers constructing an Agent (platform + model + processors),
  declaring a tool with the AsTool attribute and wiring it through a Toolbox and
  AgentProcessor, the tool-calling flow, runtime tool customization with a
  decorating Toolbox, gating tool execution via the ToolCallRequested event, and
  routing across specialist agents with MultiAgent/Handoff (including the
  Subagent tool). Triggers when creating a Symfony AI Agent, adding or
  customizing tools, requiring approval before a tool runs, or coordinating
  multiple agents.
---

# Symfony AI — Agents & Tool Calling

The Agent component (`symfony/ai-agent`) sits on top of the Platform component
and lets an LLM call your PHP code ("tools"). Install with
`composer require symfony/ai-agent` (plus `symfony/ai-platform`).

## Constructing an Agent

`Symfony\AI\Agent\Agent` wraps a `PlatformInterface` and a model name. The full
constructor is:

```php
new Agent(
    PlatformInterface $platform,
    string $model,              // model name, e.g. 'gpt-4o-mini'
    iterable $inputProcessors = [],
    iterable $outputProcessors = [],
    string $name = 'agent',     // used by multi-agent routing
);
```

Run it with a `MessageBag` and optional per-call options:

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$agent = new Agent($platform, 'gpt-4o-mini');
$messages = new MessageBag(
    Message::forSystem('You are a helpful assistant.'),
    Message::ofUser('Hello!'),
);
$result = $agent->call($messages);
echo $result->getContent();
```

Tools are NOT enabled by passing them to the constructor — they go through a
**Toolbox** and an **AgentProcessor** registered as input AND output processor
(see below). Per-call options (`['stream' => true]`, `['tools' => [...]]`,
`['use_memory' => false]`) are passed as the second argument to `call()`.

## Defining a tool

Any class becomes a tool by adding the
`Symfony\AI\Agent\Toolbox\Attribute\AsTool` attribute. Its arguments are
`name`, `description`, and an optional `method` (defaults to `__invoke`). The
LLM sees the name + description; parameter descriptions come from `@param`
docblocks on the called method.

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool('weather', 'Fetches the current weather for a given city')]
final class WeatherTool
{
    /**
     * @param string $city  The name of the city to look up
     * @param string $units "metric" or "imperial"
     */
    public function __invoke(string $city, string $units = 'metric'): array
    {
        return ['city' => $city, 'temperature' => '22°C', 'condition' => 'sunny'];
    }
}
```

Rules:

- A tool may return a string, array, object, scalar, or `DateTimeInterface` —
  `ToolResultConverter` serializes non-strings to JSON, so you rarely call
  `json_encode()` yourself.
- Repeat `#[AsTool(..., method: 'current')]` to expose **multiple tools per
  class**, one per method.
- Tools are plain services, so inject dependencies (repositories, HTTP clients)
  via the constructor.

Parameter schema, validation (`#[Schema]`, Symfony Validator, native enums),
third-party tools (`MemoryToolFactory`/`ChainFactory`), fault tolerance, and
tool sources are covered in
[references/tools-and-schema.md](references/tools-and-schema.md).

## Wiring tools into the agent (the tool-calling flow)

Put tool instances in a `Toolbox`, wrap it in an `AgentProcessor`, and register
that processor as both the input and output processor:

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;

$toolbox = new Toolbox([new WeatherTool()]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

$result = $agent->call($messages);
```

On `call()`, the agent advertises the toolbox's tools to the LLM. If the model
requests tool calls, the toolbox executes them and feeds results back to the
model, looping until a final answer. Key controls:

- **`maxToolCalls`** (AgentProcessor) — cap the number of tool calls to limit
  cost and prevent infinite loops.
- **`['tools' => ['weather']]`** option on `call()` — restrict a single call to
  a subset of configured tools by name.
- **`excludeToolMessages: true`** (AgentProcessor) — drop tool request/result
  messages from the `MessageBag` afterwards.

## Dynamic tools (runtime add / remove / rename)

To change the tool set at runtime, decorate the toolbox: implement
`Symfony\AI\Agent\Toolbox\ToolboxInterface` (methods `getTools()` and
`execute(ToolCall): ToolResult`) and delegate to the inner toolbox. In the AI
Bundle use `#[AsDecorator('ai.toolbox.<agent>')]` with `#[AutowireDecorated]` to
wrap an existing agent's toolbox.

- **Rename / re-describe**: in `getTools()`, replace the matching
  `Symfony\AI\Platform\Tool\Tool` with a new one carrying a different name or
  description (reuse `$tool->getReference()` and `$tool->getParameters()`).
- **Remove**: `array_filter()` it out of `getTools()` (e.g. behind a feature
  toggle).
- **Add**: append a new `Tool` (with an `ExecutionReference` and a JSON-Schema
  parameter array), then handle its name in `execute()` before delegating.

Full decorator example in
[references/dynamic-tools.md](references/dynamic-tools.md).

## Human-in-the-loop approval

The `Toolbox` dispatches a
`Symfony\AI\Agent\Toolbox\Event\ToolCallRequested` event **before each tool
runs**. A listener inspects the call and chooses one of three outcomes:

- **Allow** — do nothing; the tool executes.
- **Deny** — `$event->deny($reason)`; execution is blocked and the reason is
  returned to the LLM.
- **Replace** — `$event->setResult($result)` with a `ToolResult`; execution is
  skipped and your custom result is returned.

Pass the dispatcher to the toolbox via its `eventDispatcher:` argument:

```php
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event): void {
    if ('delete_account' === $event->getToolCall()->getName()) {
        $event->deny('Account deletion requires manual approval.');
    }
});

$toolbox = new Toolbox($tools, eventDispatcher: $dispatcher);
```

`$event->getToolCall()` exposes the name and arguments; `$event->getMetadata()`
returns the tool's `Tool` (description + parameter schema) for richer prompts. In
a Symfony app, register the listener with `#[AsEventListener]` — the event is
inferred from the `__invoke()` argument. A policy + confirmation-handler pattern
(auto-allow reads, ask on writes, remember decisions) is in
[references/human-in-the-loop.md](references/human-in-the-loop.md).

> Other lifecycle events — `ToolCallArgumentsResolved`, `ToolCallSucceeded`,
> `ToolCallFailed`, and `ToolCallsExecuted` (react to all results at once) — are
> available for observability and result interception.

## Multi-agent orchestration

Two ways one agent can use another:

**1. Subagent as a tool.** Wrap an agent in
`Symfony\AI\Agent\Toolbox\Tool\Subagent` and register it like a third-party tool
so the outer LLM can call it (useful to hide internals or reuse an agent):

```php
use Symfony\AI\Agent\Toolbox\Tool\Subagent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\ToolFactory\MemoryToolFactory;

$subagent = new Subagent($researchAgent);
$factory = (new MemoryToolFactory())
    ->addTool($subagent, 'research_agent', 'Performs in-depth research');
$toolbox = new Toolbox([$subagent], $factory);
```

**2. Orchestrator + handoffs.** For routing to specialists, give each
specialist `Agent` a distinct `name` and a
`Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor`, then assemble a
`Symfony\AI\Agent\MultiAgent\MultiAgent` from an orchestrator, an array of
`Handoff`s, and a fallback agent:

```php
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;

$handoffs = [
    new Handoff(to: $technical, when: ['bug', 'error', 'exception']),
    new Handoff(to: $billing,   when: ['invoice', 'payment', 'billing']),
];

$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,  // routes; needs no name
    handoffs: $handoffs,          // at least one required
    fallback: $fallback,          // handles unmatched questions
);

$result = $multiAgent->call($messages); // called like any agent
```

A `Handoff` requires a target agent (`to:`) and a non-empty `when:` keyword
array; `MultiAgent` requires at least one handoff. The orchestrator analyzes
each question and delegates to the matching specialist, falling back when none
match. Pass a PSR-3 logger to the `MultiAgent` constructor to trace routing.
Full specialist/orchestrator setup in
[references/multi-agent.md](references/multi-agent.md).
