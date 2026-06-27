# Human-in-the-loop tool confirmation

Companion to `SKILL.md`. Builds on the
`Symfony\AI\Agent\Toolbox\Event\ToolCallRequested` event, which fires before
every tool execution. A listener can **allow** (do nothing), **deny**
(`$event->deny($reason)`), or **replace** (`$event->setResult($result)`).

## Confirm every call (CLI)

```php
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event): void {
    $toolCall = $event->getToolCall();
    echo sprintf(
        "Tool '%s' wants to run with args: %s\nAllow? [y/N] ",
        $toolCall->getName(),
        json_encode($toolCall->getArguments()),
    );

    if ('y' !== strtolower(trim(fgets(\STDIN)))) {
        $event->deny('User denied tool execution.');
    }
});

$toolbox = new Toolbox($tools, eventDispatcher: $dispatcher);
```

## Add a policy (don't ask for everything)

Decide per tool whether to allow, deny, or ask:

```php
enum PolicyDecision { case Allow; case Deny; case AskUser; }

interface PolicyInterface
{
    public function decide(ToolCall $toolCall): PolicyDecision;
}

use Symfony\AI\Platform\Result\ToolCall;

class ReadAllowPolicy implements PolicyInterface
{
    public function decide(ToolCall $toolCall): PolicyDecision
    {
        $name = strtolower($toolCall->getName());
        foreach (['read', 'get', 'list', 'search', 'find', 'show'] as $pattern) {
            if (str_contains($name, $pattern)) {
                return PolicyDecision::Allow;
            }
        }

        return PolicyDecision::AskUser;
    }
}
```

## Confirmation handler

The handler prompts the user; its shape depends on context (CLI, HTTP, async):

```php
use Symfony\AI\Platform\Result\ToolCall;

class CliConfirmationHandler
{
    public function confirm(ToolCall $toolCall): bool
    {
        echo sprintf(
            "Allow tool '%s' with args %s? [y/N] ",
            $toolCall->getName(),
            json_encode($toolCall->getArguments()),
        );

        return 'y' === strtolower(trim(fgets(\STDIN)));
    }
}
```

For web apps, store pending confirmations and resolve them via an HTTP endpoint
or WebSocket. `$event->getMetadata()` exposes the tool's description and
parameter schema for richer prompts.

## Wire policy + handler

```php
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) use ($policy, $handler): void {
    $decision = $policy->decide($event->getToolCall());

    if (PolicyDecision::Allow === $decision) {
        return;
    }
    if (PolicyDecision::Deny === $decision) {
        $event->deny('Tool blocked by policy.');

        return;
    }
    // AskUser
    if (!$handler->confirm($event->getToolCall())) {
        $event->deny('User denied tool execution.');
    }
});
```

## Remember decisions (always / never)

Cache per-tool answers so the user is not asked repeatedly:

```php
$decisions = [];
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) use (&$decisions): void {
    $name = $event->getToolCall()->getName();
    if (isset($decisions[$name])) {
        if (!$decisions[$name]) {
            $event->deny('Tool previously denied by user.');
        }

        return;
    }

    echo sprintf("Allow tool '%s'? [y/N/always/never] ", $name);
    $input = strtolower(trim(fgets(\STDIN)));
    $allowed = in_array($input, ['y', 'always'], true);

    if (in_array($input, ['always', 'never'], true)) {
        $decisions[$name] = $allowed;
    }
    if (!$allowed) {
        $event->deny('User denied tool execution.');
    }
});
```

## Register the listener in Symfony

`#[AsEventListener]` auto-registers the listener; the event is inferred from the
`__invoke()` argument — no service config needed:

```php
namespace App\EventListener;

use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
class ToolConfirmationListener
{
    public function __invoke(ToolCallRequested $event): void
    {
        // confirmation logic
    }
}
```
