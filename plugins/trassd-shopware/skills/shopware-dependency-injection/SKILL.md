---
name: shopware-dependency-injection
description: >-
  Register and customize services in a Shopware 6 plugin — service config file
  (Resources/config/services.php or .xml), autowiring/autoconfigure, constructor
  injection, explicit arguments, common service tags, and decorating/adjusting an
  existing core service (Symfony service decoration with the inner reference).
  Triggers when adding a custom service, editing a plugin's services config,
  injecting one service into another, or decorating/overriding a core service.
---

# Shopware Dependency Injection

Shopware uses the Symfony [DI container](https://symfony.com/doc/current/service_container.html).
A plugin declares its services in a config file under `Resources/config/`; the
plugin base class loads it automatically. Use this skill when creating a service,
wiring dependencies, tagging a service, or decorating an existing one.

## Where the config file lives and how it loads

- Put the service config in `<plugin root>/src/Resources/config/`.
- The plugin's `Bundle` base class auto-discovers and loads any
  `Resources/config/services.*` file — no manual registration needed. The
  delegating loader supports **PHP** (`services.php`), **XML** (`services.xml`),
  and **YAML** (`services.yaml`). A matching `services_test.*` is additionally
  loaded in the `test` environment.
- **Format choice:** the docs recommend the **PHP** configurator
  (`services.php`). XML still works and is used throughout the Shopware core, but
  Symfony deprecated XML service config as of Symfony 7.4 (to be removed in
  Symfony 8.0). Prefer PHP for new plugins; keep XML edits consistent with the
  file already in the plugin.

## Defining a service (PHP configurator)

```php
// <plugin root>/src/Resources/config/services.php
<?php declare(strict_types=1);

use Swag\BasicExample\Service\ExampleService;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (ContainerConfigurator $configurator): void {
    $services = $configurator->services();

    $services->set(ExampleService::class);
};
```

By default **all services in Shopware 6 are private**. Only mark a service public
when something resolves it from the container by id directly.

## Autowiring vs. explicit arguments

**Autowire + autoconfigure** — register everything in `src/` by type and let
Symfony resolve constructor dependencies and apply tags automatically:

```php
$services = $configurator->services()
    ->defaults()
        ->autowire()
        ->autoconfigure();

$services->load('Swag\\BasicExample\\', '../../')
    ->exclude('../../{Resources,Migration,*.php}');
```

`Resources` and `Migration` are excluded because they normally hold no services.

**Explicit declaration** — register a single service and pass its arguments by
hand when you want full control. Inject another service with the `service()`
helper:

```php
use Shopware\Core\System\SystemConfig\SystemConfigService;
use function Symfony\Component\DependencyInjection\Loader\Configurator\service;

$services->set(ExampleService::class)
    ->args([service(SystemConfigService::class)]);
```

With autowiring enabled, no `args()` line is needed — Symfony injects
`SystemConfigService` from the constructor type-hint automatically.

## Constructor injection

Inject dependencies through the constructor (use promoted properties):

```php
namespace Swag\BasicExample\Service;

use Shopware\Core\System\SystemConfig\SystemConfigService;

class ExampleService
{
    public function __construct(
        private SystemConfigService $systemConfigService
    ) {
    }
}
```

## Common service tags

Tags integrate a service into a Shopware/Symfony extension point. With
`autoconfigure()` many are applied automatically from the implemented
interface/base class; otherwise add them explicitly. Tags seen in the Shopware
core include:

- `shopware.entity.definition` — register a DAL entity definition.
- `console.command` — register a CLI command.
- `kernel.event_subscriber` — register an event subscriber.
- `messenger.message_handler` — register a message-queue handler.
- `shopware.scheduled.task` — register a scheduled task.

## Decorating / adjusting an existing service

To change behavior of a core or third-party service without replacing it, use
Symfony **service decoration**: define your service, mark it with `decorate()`
pointing at the target, and pass the original in via the `.inner` reference.

```php
$services->set(ExampleServiceDecorator::class)
    ->decorate(ExampleService::class)
    ->args([service('.inner')]);
```

The decorator receives the original service in its constructor, calls into it,
and augments the result. Shopware's recommended pattern uses an **abstract base
class** (e.g. `AbstractExampleService`) with an abstract `getDecorated()` method
instead of an interface, so new methods can be added without breaking other
decorators; the undecorated implementation throws `DecorationPatternException`
from `getDecorated()`.

Full end-to-end example (abstract class, original service, decorator, and the
backwards-compatible "add a new method" pattern): see
[references/decorating-a-service.md](references/decorating-a-service.md).

## Rules

- Service config goes in `src/Resources/config/services.*`; it is loaded
  automatically — never wire it up manually in the plugin class.
- Prefer the PHP configurator for new plugins; XML is deprecated upstream but
  still valid for existing files.
- Keep services private unless an external lookup truly needs them public.
- Prefer constructor injection; prefer autowiring for plugin-internal services
  and explicit `args()` only when you need control.
- To change existing behavior, decorate — do not copy/replace the original.
- Reference the original inside a decorator with `service('.inner')` and expose
  it through `getDecorated()`.
