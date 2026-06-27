# Scheduled tasks — registration, run loop, debugging

## File placement

Both classes and their registration live under the plugin's `src/`:

- Task + handler: e.g. `src/Service/ScheduledTask/ExampleTask.php` and
  `ExampleTaskHandler.php` (the directory name is a convention, not a
  requirement).
- DI registration: `src/Resources/config/services.php`, which Shopware
  autoloads when it sits in `Resources/config/`.

## Full services.php registration

```php
<?php declare(strict_types=1);

use Swag\Example\ScheduledTask\CleanupTask;
use Swag\Example\ScheduledTask\CleanupTaskHandler;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

use function Symfony\Component\DependencyInjection\Loader\Configurator\service;

return static function (ContainerConfigurator $configurator): void {
    $services = $configurator->services();

    $services->set(CleanupTask::class)
        ->tag('shopware.scheduled.task');

    $services->set(CleanupTaskHandler::class)
        ->args([
            service('scheduled_task.repository'),
            service('logger'),
        ])
        ->tag('messenger.message_handler');
};
```

The task requires the `shopware.scheduled.task` tag; the handler requires the
`messenger.message_handler` tag.

## Registration lifecycle

Scheduled tasks are normally written into the `scheduled_task` database table
when the plugin is installed or updated. To register a newly added task without
reinstalling the plugin:

```bash
bin/console scheduled-task:register
```

## Running tasks

Two steps are needed when not relying on the admin worker:

1. Start the runner, which dispatches a task message to the bus once an
   interval is due:

   ```bash
   bin/console scheduled-task:run
   ```

2. Consume the dispatched messages to actually execute the handlers:

   ```bash
   bin/console messenger:consume
   ```

The task's `status` in the `scheduled_task` table must be `scheduled` for it to
run. With the admin worker enabled, the manual consume step is unnecessary.

## Notes grounded in core

- `ScheduledTask` is an abstract class that itself implements
  `AsyncMessageInterface` — that is why its handler is registered as a message
  handler rather than via a bespoke mechanism.
- `getTaskName()` and `getDefaultInterval()` are abstract and must be
  implemented; `ScheduledTaskHandler::run()` is abstract and must be
  implemented.
