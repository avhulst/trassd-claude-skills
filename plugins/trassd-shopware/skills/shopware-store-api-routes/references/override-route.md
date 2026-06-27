# Overriding an existing Store-API route via decoration

To add data to (or otherwise extend) an existing route, decorate it. The
decorator extends the **same abstract class** as the original, wraps the inner
route, and delegates to it — it must never re-implement the original's logic.

## Decorator class

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Core\Content\Example\SalesChannel;

use Shopware\Core\Framework\DataAbstractionLayer\EntityRepository;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Core\Framework\Routing\StoreApiRouteScope;
use Shopware\Core\PlatformRequest;
use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Symfony\Component\Routing\Attribute\Route;

#[Route(defaults: [PlatformRequest::ATTRIBUTE_ROUTE_SCOPE => [StoreApiRouteScope::ID]])]
class ExampleRouteDecorator extends AbstractExampleRoute
{
    public function __construct(
        private readonly EntityRepository $exampleRepository,
        private readonly AbstractExampleRoute $decorated,
    ) {
    }

    public function getDecorated(): AbstractExampleRoute
    {
        return $this->decorated;
    }

    #[Route(path: '/store-api/example', name: 'store-api.example.search', methods: ['GET', 'POST'])]
    public function load(Criteria $criteria, SalesChannelContext $context): ExampleRouteResponse
    {
        // Always delegate to the inner route — this preserves the contract.
        $response = $this->decorated->load($criteria, $context);

        // Augment the response without breaking existing callers, e.g. headers:
        $response->headers->set('cache-control', 'max-age=10000');

        return $response;
    }
}
```

Rules:
- Extend the abstract route, accept an `AbstractExampleRoute` in the
  constructor, return it from `getDecorated()`.
- Delegate to the inner route, then **add** data/headers — do not remove or
  rename existing fields (backwards compatibility).
- The response stays a `StoreApiResponse` subclass with a single object.

## Service registration

Register the decorator **after** the original, using `->decorate()` pointing at
the original and `service('.inner')` for the wrapped instance.

```php
<?php declare(strict_types=1);

use Swag\BasicExample\Core\Content\Example\SalesChannel\ExampleRoute;
use Swag\BasicExample\Core\Content\Example\SalesChannel\ExampleRouteDecorator;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

use function Symfony\Component\DependencyInjection\Loader\Configurator\service;

return static function (ContainerConfigurator $configurator): void {
    $services = $configurator->services();

    $services->set(ExampleRouteDecorator::class)
        ->decorate(ExampleRoute::class)
        ->public()
        ->args([
            service('swag_example.repository'),
            service('.inner'),
        ]);
};
```

The same approach decorates **core** Store-API routes: extend the core route's
abstract class, inject the core abstract route, decorate the core service, and
delegate.
