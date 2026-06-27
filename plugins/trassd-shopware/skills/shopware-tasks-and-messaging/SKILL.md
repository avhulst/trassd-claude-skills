---
name: shopware-tasks-and-messaging
description: >-
  Run background work in a Shopware 6 plugin — scheduled tasks, custom console
  commands, the message queue (messages, handlers, middleware), and logging.
  Use when adding a ScheduledTask, a Symfony console command, a message and its
  handler, dispatching to the message bus, or wiring a plugin Monolog channel.
---

# Shopware Tasks & Messaging

Background processing in a Shopware 6 plugin builds on the Symfony Messenger
component. Scheduled tasks, the message queue, and async handlers all flow
through the message bus. This skill covers the four building blocks: scheduled
tasks, console commands, the message queue, and logging.

All service registration happens in `<plugin root>/src/Resources/config/services.php`,
which Shopware autoloads. Keep `getTaskName()` and message classes vendor-prefixed
to avoid collisions with core or other plugins.

## Scheduled tasks

A scheduled task is two classes plus DI registration.

1. **The task** extends `Shopware\Core\Framework\MessageQueue\ScheduledTask\ScheduledTask`
   and implements two static methods. (Under the hood `ScheduledTask` is itself a
   message — it implements `AsyncMessageInterface` — which is why the handler is a
   message handler.)

   ```php
   namespace Swag\Example\ScheduledTask;

   use Shopware\Core\Framework\MessageQueue\ScheduledTask\ScheduledTask;

   class CleanupTask extends ScheduledTask
   {
       public static function getTaskName(): string
       {
           return 'swag.cleanup_task'; // vendor-prefixed, unique
       }

       public static function getDefaultInterval(): int
       {
           return 300; // seconds
       }
   }
   ```

2. **The handler** extends `...\ScheduledTask\ScheduledTaskHandler`, is annotated
   with `#[AsMessageHandler(handles: CleanupTask::class)]`, and implements `run()`.

   ```php
   use Shopware\Core\Framework\MessageQueue\ScheduledTask\ScheduledTaskHandler;
   use Symfony\Component\Messenger\Attribute\AsMessageHandler;

   #[AsMessageHandler(handles: CleanupTask::class)]
   class CleanupTaskHandler extends ScheduledTaskHandler
   {
       public function run(): void
       {
           // do the work
       }
   }
   ```

3. **Register both** in `services.php`. Tag the task `shopware.scheduled.task` and
   the handler `messenger.message_handler`:

   ```php
   $services->set(CleanupTask::class)->tag('shopware.scheduled.task');
   $services->set(CleanupTaskHandler::class)
       ->args([service('scheduled_task.repository'), service('logger')])
       ->tag('messenger.message_handler');
   ```

The task is saved to the `scheduled_task` table on plugin activation; force
re-registration with `bin/console scheduled-task:register`. See
[references/scheduled-tasks.md](references/scheduled-tasks.md) for the run loop
(`scheduled-task:run` + `messenger:consume`) and status notes.

## Custom console commands

Shopware CLI commands are plain Symfony Console commands. Register the command
class as a service and tag it `console.command`; it then runs via `bin/console`.

```php
$services->set(Swag\Example\Command\ExampleCommand::class)
    ->tag('console.command');
```

No Shopware-specific base class is required — follow the standard Symfony
`Command` approach.

## Message queue

Shopware integrates Symfony Messenger. A **message** is a serializable PHP object
carrying everything its handler needs; the bus wraps it in an envelope when
dispatched. Messages are handled **synchronously by default**.

- **Async:** implement `Shopware\Core\Framework\MessageQueue\AsyncMessageInterface`.
- **Lower priority async:** implement
  `Shopware\Core\Framework\MessageQueue\LowPriorityMessageInterface`
  (`low_priority` queue; requires Shopware 6.5.7.0+).

```php
use Shopware\Core\Framework\MessageQueue\AsyncMessageInterface;

class SmsNotification implements AsyncMessageInterface
{
    public function __construct(private readonly string $content) {}
    public function getContent(): string { return $this->content; }
}
```

**Dispatch** by injecting `Symfony\Component\Messenger\MessageBusInterface`
(service `messenger.default_bus`) and calling `dispatch()`:

```php
public function __construct(private readonly MessageBusInterface $bus) {}

public function send(string $text): void
{
    $this->bus->dispatch(new SmsNotification($text));
}
```

**Handle** with a class marked `#[AsMessageHandler]` and an `__invoke()` typed to
the message; register it with the `messenger.message_handler` tag. Multiple
handlers may handle the same message.

```php
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class SmsHandler
{
    public function __invoke(SmsNotification $message): void
    {
        // process the message
    }
}
```

For envelope stamps (e.g. `DelayStamp`), per-message transport routing
(`routing_overwrite` in `shopware.yaml`), and middleware notes, see
[references/message-queue.md](references/message-queue.md).

## Plugin logging

Give your plugin its own Monolog channel so project owners can redirect it.

1. Make the plugin load `Resources/config/packages/*.yaml` (a Symfony bundle
   requirement — wire a config loader in the plugin base class `build()`, or use a
   bundle extension).
2. Declare a uniquely named channel, and optionally a handler, in
   `Resources/config/packages/monolog.yaml`:

   ```yaml
   monolog:
     channels: ['my_plugin_channel']
     handlers:
       myPluginLogHandler:
         type: rotating_file
         path: "%kernel.logs_dir%/my_plugin_%kernel.environment%.log"
         level: error
         channels: ["my_plugin_channel"]
   ```

3. Inject the auto-registered logger via service ID
   `monolog.logger.my_plugin_channel`.

See [references/logging.md](references/logging.md) for the full base-class config
loader snippet.

## Checklist

- Scheduled task: extends `ScheduledTask`, implements `getTaskName()` (prefixed)
  + `getDefaultInterval()`; handler extends `ScheduledTaskHandler`, has
  `#[AsMessageHandler(handles: …)]` + `run()`.
- Both registered with the correct tags (`shopware.scheduled.task`,
  `messenger.message_handler`); task re-registered if needed.
- Console command tagged `console.command`.
- Message implements `AsyncMessageInterface` if it must run async; dispatched via
  `MessageBusInterface`; handler tagged `messenger.message_handler`.
- Logging uses a dedicated, uniquely named Monolog channel, not the default.
