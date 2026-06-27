# Decorating a service — full example

This walks through the complete Shopware-recommended decoration pattern: an
abstract base class, the original service, and the decorator. It mirrors the
"Adjusting a Service" guide. Replace `ExampleService` with the core or
third-party service you actually want to adjust — the abstract class and base
implementation usually already exist in the code you are extending and are shown
here only so the example is self-contained.

## 1. Register the services

```php
// <plugin root>/src/Resources/config/services.php
<?php declare(strict_types=1);

use Swag\BasicExample\Service\ExampleService;
use Swag\BasicExample\Service\ExampleServiceDecorator;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

use function Symfony\Component\DependencyInjection\Loader\Configurator\service;

return static function (ContainerConfigurator $configurator): void {
    $services = $configurator->services();

    $services->set(ExampleService::class);

    $services->set(ExampleServiceDecorator::class)
        ->decorate(ExampleService::class)
        ->args([service('.inner')]);
};
```

`decorate()` points at the service being wrapped; `service('.inner')` injects the
original (pre-decoration) instance into the decorator.

## 2. Abstract base class

An abstract class (rather than an interface) is recommended so new methods can be
added incrementally without breaking existing decorators. It must declare an
abstract `getDecorated()` returning the abstract type.

```php
// <plugin root>/src/Service/AbstractExampleService.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Service;

abstract class AbstractExampleService
{
    abstract public function getDecorated(): AbstractExampleService;

    abstract public function doSomething(): string;
}
```

## 3. Original implementation

The undecorated implementation has no decorated instance, so its `getDecorated()`
throws `DecorationPatternException`.

```php
// <plugin root>/src/Service/ExampleService.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Service;

use Shopware\Core\Framework\Plugin\Exception\DecorationPatternException;

class ExampleService extends AbstractExampleService
{
    public function getDecorated(): AbstractExampleService
    {
        throw new DecorationPatternException(self::class);
    }

    public function doSomething(): string
    {
        return 'Did something.';
    }
}
```

## 4. Decorator

Extends the abstract class, accepts an instance of it in the constructor, returns
that instance from `getDecorated()`, and augments the wrapped result.

```php
// <plugin root>/src/Service/ExampleServiceDecorator.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Service;

class ExampleServiceDecorator extends AbstractExampleService
{
    private AbstractExampleService $decoratedService;

    public function __construct(AbstractExampleService $exampleService)
    {
        $this->decoratedService = $exampleService;
    }

    public function getDecorated(): AbstractExampleService
    {
        return $this->decoratedService;
    }

    public function doSomething(): string
    {
        $originalResult = $this->decoratedService->doSomething();

        return $originalResult . ' Did something additionally.';
    }
}
```

## Adding a new method (backwards compatible)

When a service is decorated in several places, add new methods as **normal public
methods first**, not abstract — otherwise every existing decorator breaks because
the method becomes required everywhere. The default implementation in the
abstract class delegates through `getDecorated()`:

```php
// AbstractExampleService.php
abstract class AbstractExampleService
{
    abstract public function getDecorated(): AbstractExampleService;

    abstract public function doSomething(): string;

    public function doSomethingNew(): string
    {
        return $this->getDecorated()->doSomethingNew();
    }
}
```

```php
// ExampleService.php — implement the new method in the base service
public function doSomethingNew(): string
{
    return 'Did something new.';
}
```

Once every decorator implements the method, it can be promoted to `abstract` in a
future release.
