# Multi-agent orchestration

Companion to `SKILL.md`. Build a system where a central orchestrator routes each
question to a specialist agent, with a fallback for unmatched questions.

## Specialist agents

Each specialist is a regular `Agent` with a `SystemPromptInputProcessor`
defining its domain, plus a distinct `name` so the orchestrator can identify it:

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;

$technical = new Agent(
    $platform,
    'gpt-5-mini',
    [new SystemPromptInputProcessor(
        'You are a technical support specialist. Help users resolve bugs and errors.',
    )],
    name: 'technical',
);

$billing = new Agent(
    $platform,
    'gpt-5-mini',
    [new SystemPromptInputProcessor(
        'You are a billing specialist. Help users with invoices and payments.',
    )],
    name: 'billing',
);
```

## Orchestrator

Another agent whose prompt tells it to route. It is the entry point and needs no
name:

```php
$orchestrator = new Agent(
    $platform,
    'gpt-5-mini',
    [new SystemPromptInputProcessor(
        'You are an agent orchestrator that routes user questions to specialized agents.',
    )],
);
```

## Handoffs

A `Symfony\AI\Agent\MultiAgent\Handoff` maps trigger keywords to a target agent.
`to:` is the destination agent; `when:` is a non-empty keyword array (empty
throws):

```php
use Symfony\AI\Agent\MultiAgent\Handoff;

$handoffs = [
    new Handoff(to: $technical, when: ['bug', 'error', 'exception', 'technical']),
    new Handoff(to: $billing,   when: ['invoice', 'payment', 'billing', 'subscription']),
];
```

## Assemble the MultiAgent

`Symfony\AI\Agent\MultiAgent\MultiAgent` needs an orchestrator, at least one
handoff, and a fallback agent. A PSR-3 logger (optional) traces which agent
handles each request:

```php
use Symfony\AI\Agent\MultiAgent\MultiAgent;

$fallback = new Agent(
    $platform,
    'gpt-5-mini',
    [new SystemPromptInputProcessor(
        'You are a general assistant. Help users with any non-specialized questions.',
    )],
    name: 'fallback',
);

$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: $handoffs,
    fallback: $fallback,
    // logger: $psrLogger,
);
```

## Route questions

Call it like any agent; routing is automatic:

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$messages = new MessageBag(
    Message::ofUser('I get a "Call to undefined method" error in my controller.'),
);
$result = $multiAgent->call($messages); // -> technical specialist

$messages = new MessageBag(
    Message::ofUser('Can you recommend a good pasta recipe?'),
);
$result = $multiAgent->call($messages); // -> fallback
```

Each handoff is evaluated independently, so you can add as many specialists as
needed.

## Alternative: Subagent as a tool

When you want the outer LLM to invoke another agent as a tool (rather than route
to it), wrap it in `Symfony\AI\Agent\Toolbox\Tool\Subagent` and register it as a
named tool through a `MemoryToolFactory` (see `SKILL.md`). `Subagent` takes a
single `AgentInterface` and forwards the message string to it via `__invoke()`.
