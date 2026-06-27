# Custom Storefront controller — full wiring

A controller is just a service registered in the container. Four files are
involved: the controller, `services.php`, `routes.php`, and the template.

## Controller class

```php
// PLUGIN_ROOT/src/Storefront/Controller/ExampleController.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Storefront\Controller;

use Shopware\Core\PlatformRequest;
use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Shopware\Storefront\Controller\StorefrontController;
use Shopware\Storefront\Framework\Routing\StorefrontRouteScope;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

#[Route(defaults: [PlatformRequest::ATTRIBUTE_ROUTE_SCOPE => [StorefrontRouteScope::ID]])]
class ExampleController extends StorefrontController
{
    #[Route(path: '/example', name: 'frontend.example.example', methods: ['GET'])]
    public function showExample(Request $request, SalesChannelContext $context): Response
    {
        return $this->renderStorefront('@SwagBasicExample/storefront/page/example.html.twig', [
            'example' => 'Hello world',
        ]);
    }
}
```

Notes:

- The `_routeScope` default must be present (class-level, as above, or repeated
  on each method route). The scope value is `StorefrontRouteScope::ID`. Prior to
  6.4.11.0 this used the deprecated `@RouteScope` annotation — do not use it.
- Route `name` must start with `frontend`, `widgets`, or `payment`. Other
  prefixes won't be treated as Storefront routes unless allowed explicitly
  (see below).
- `renderStorefront` returns a `Response`; every routed action must return one.
- `Request` and `SalesChannelContext` are injected as method arguments when
  needed.

## services.php

Controllers must be `public()` and receive the container via `setContainer`:

```php
// PLUGIN_ROOT/src/Resources/config/services.php
<?php declare(strict_types=1);

use Swag\BasicExample\Storefront\Controller\ExampleController;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

use function Symfony\Component\DependencyInjection\Loader\Configurator\service;

return static function (ContainerConfigurator $configurator): void {
    $services = $configurator->services();

    $services->set(ExampleController::class)
        ->public()
        ->call('setContainer', [service('service_container')]);
};
```

When the controller depends on a page loader, inject it via `->args([...])`
(constructor injection into a private property).

## routes.php

Tell Shopware where to scan for route attributes:

```php
// PLUGIN_ROOT/src/Resources/config/routes.php
<?php declare(strict_types=1);

use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;

return static function (RoutingConfigurator $routes): void {
    $routes->import('../../Storefront/Controller/*Controller.php', 'attribute');
};
```

## Template

The rendered view recreates the path under `Resources/views`:

```twig
{# PLUGIN_ROOT/src/Resources/views/storefront/page/example.html.twig #}
{% sw_extends '@Storefront/storefront/base.html.twig' %}

{% block base_content %}
    <h1>Our example controller!</h1>
{% endblock %}
```

## Allowing custom route names (since 6.7.2.0)

To use a route name without the `frontend`/`widgets`/`payment` prefix, add a
config file and ensure it is loaded during container build:

```yaml
# PLUGIN_ROOT/src/Resources/config/packages/storefront.yaml
storefront:
    router:
        allowed_routes:
            - swag.test.foo-bar
```

Override the plugin base class `build()` to load `Resources/config/packages/*.yaml`
(via a `DelegatingLoader` with `YamlFileLoader` / `GlobFileLoader` /
`DirectoryLoader`). The route name `swag.test.foo-bar` is then usable directly.
