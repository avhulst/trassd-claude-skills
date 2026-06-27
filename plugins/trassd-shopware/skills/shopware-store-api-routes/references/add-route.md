# Adding a Store-API route — full sample

Layout used here (a route for entity `swag_example` exposed at `/store-api/example`):

```
src/Core/Content/Example/SalesChannel/AbstractExampleRoute.php
src/Core/Content/Example/SalesChannel/ExampleRoute.php
src/Core/Content/Example/SalesChannel/ExampleRouteResponse.php
src/Resources/config/services.php
src/Resources/config/routes.php
```

## Abstract route

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Core\Content\Example\SalesChannel;

use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Core\System\SalesChannel\SalesChannelContext;

abstract class AbstractExampleRoute
{
    abstract public function getDecorated(): AbstractExampleRoute;

    abstract public function load(Criteria $criteria, SalesChannelContext $context): ExampleRouteResponse;
}
```

## Concrete route

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Core\Content\Example\SalesChannel;

use Shopware\Core\Framework\DataAbstractionLayer\EntityRepository;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Core\Framework\Plugin\Exception\DecorationPatternException;
use Shopware\Core\Framework\Routing\StoreApiRouteScope;
use Shopware\Core\PlatformRequest;
use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Symfony\Component\Routing\Attribute\Route;

#[Route(defaults: [PlatformRequest::ATTRIBUTE_ROUTE_SCOPE => [StoreApiRouteScope::ID]])]
class ExampleRoute extends AbstractExampleRoute
{
    public function __construct(private readonly EntityRepository $exampleRepository)
    {
    }

    public function getDecorated(): AbstractExampleRoute
    {
        throw new DecorationPatternException(self::class);
    }

    #[Route(path: '/store-api/example', name: 'store-api.example.search', methods: ['GET', 'POST'], defaults: ['_entity' => 'swag_example'])]
    public function load(Criteria $criteria, SalesChannelContext $context): ExampleRouteResponse
    {
        return new ExampleRouteResponse($this->exampleRepository->search($criteria, $context->getContext()));
    }
}
```

## Response struct

Extends `StoreApiResponse`; the inherited `$object` (an `EntitySearchResult`)
is the single object the route returns.

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Core\Content\Example\SalesChannel;

use Shopware\Core\Framework\DataAbstractionLayer\Search\EntitySearchResult;
use Shopware\Core\System\SalesChannel\StoreApiResponse;
use Swag\BasicExample\Core\Content\Example\ExampleCollection;

/**
 * @property EntitySearchResult<ExampleCollection> $object
 */
class ExampleRouteResponse extends StoreApiResponse
{
    public function getExamples(): ExampleCollection
    {
        return $this->object->getEntities();
    }
}
```

## Service registration

```php
<?php declare(strict_types=1);

use Swag\BasicExample\Core\Content\Example\SalesChannel\ExampleRoute;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

use function Symfony\Component\DependencyInjection\Loader\Configurator\service;

return static function (ContainerConfigurator $configurator): void {
    $services = $configurator->services();

    $services->set(ExampleRoute::class)
        ->args([service('swag_example.repository')]);
};
```

## Route import (attribute routing)

```php
<?php declare(strict_types=1);

use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;

return static function (RoutingConfigurator $routes): void {
    $routes->import('../../Core/**/*Route.php', 'attribute');
    // also import Storefront controllers when wrapping the route for the storefront:
    $routes->import('../../Storefront/**/*Controller.php', 'attribute');
};
```

Check registration: `./bin/console debug:router store-api.example.search`.
Document the endpoint (optional) with an OpenAPI JSON in
`src/Resources/Schema/StoreApi/`, visible at
`/store-api/_info/stoplightio.html`.

## Wrapping the route for the Storefront

A thin controller in the `storefront` scope that injects the **abstract** route
and delegates — no repository access, no business logic.

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Storefront\Controller;

use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Core\PlatformRequest;
use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Shopware\Storefront\Controller\StorefrontController;
use Shopware\Storefront\Framework\Routing\StorefrontRouteScope;
use Swag\BasicExample\Core\Content\Example\SalesChannel\AbstractExampleRoute;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

#[Route(defaults: [PlatformRequest::ATTRIBUTE_ROUTE_SCOPE => [StorefrontRouteScope::ID]])]
class ExampleController extends StorefrontController
{
    public function __construct(private readonly AbstractExampleRoute $route)
    {
    }

    #[Route(path: '/example', name: 'frontend.example.search', methods: ['GET', 'POST'], defaults: ['XmlHttpRequest' => 'true', '_entity' => 'swag_example'])]
    public function load(Criteria $criteria, SalesChannelContext $context): Response
    {
        return $this->route->load($criteria, $context);
    }
}
```

Register it alongside the route, injecting the route service and the container:

```php
$services->set(ExampleController::class)
    ->args([service(ExampleRoute::class)])
    ->call('setContainer', [service('service_container')]);
```
