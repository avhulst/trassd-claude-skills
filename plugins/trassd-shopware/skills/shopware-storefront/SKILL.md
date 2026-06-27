---
name: shopware-storefront
description: >-
  Extend the Shopware 6 Storefront — custom controllers and pages/pagelets,
  adding data to storefront pages, Twig template customization, and custom Twig
  functions. Triggers when adding a storefront controller or route, customizing
  storefront templates, or extending a storefront page with extra data.
---

# Shopware Storefront

The Storefront is the customer-facing layer. A plugin extends it through four
recurring building blocks: **controllers** (routes), **pages/pagelets** (the
data structs handed to templates), **Twig templates** (override + extend), and
**Twig extensions** (custom functions). Distill the rules below; for full
multi-file walkthroughs see the linked references.

All plugin Storefront code lives under `<plugin root>/src/Storefront/...` and
templates under `<plugin root>/src/Resources/views/storefront/...`.

## 1. Custom controller

A controller is a service. It must:

- Extend `Shopware\Storefront\Controller\StorefrontController`.
- Carry the route-scope on the **class** (set on every route, scope is
  `storefront`):
  `#[Route(defaults: [PlatformRequest::ATTRIBUTE_ROUTE_SCOPE => [StorefrontRouteScope::ID]])]`
  (`StorefrontRouteScope` lives in
  `Shopware\Storefront\Framework\Routing`). The scope may also be repeated per
  method route.
- Give each action a `#[Route(...)]` with `path`, `name`, and `methods`. Route
  names must start with `frontend`, `widgets`, or `payment` to be recognized as
  Storefront routes (since 6.7.2.0 you can allow custom names via
  `storefront.router.allowed_routes` config). Each action declares a return
  type and returns a `Response`.
- Render via `renderStorefront($view, $parameters)`, where `$view` is a
  namespaced template like `@SwagBasicExample/storefront/page/example.html.twig`.

```php
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

Register the controller in `Resources/config/services.php` as `public()` and add
`->call('setContainer', [service('service_container')])`. Tell Shopware to scan
for routes in `Resources/config/routes.php` via
`$routes->import('../../Storefront/Controller/*Controller.php', 'attribute');`.

See [references/controllers.md](references/controllers.md) for the full
services.php / routes.php / template wiring.

## Controller discipline (guideline rules)

- A Storefront controller **never contains business logic** and **never uses a
  repository directly** — fetch data through a page loader or Store API route.
- A route serves a single purpose; the function name is concise; declare the
  HTTP method explicitly.
- Inject dependencies via the constructor into private properties; declare them
  in the DI service definition.
- Routes that render a full page use a **page loader** to gather data. Pages
  whose data is identical for all customers get the `_httpCache` attribute.
- **Write** operations build their response with `createActionResponse()` (to
  allow forwards/redirects) and call a corresponding Store API route. Use
  Symfony flash bags for error reporting. Every Storefront feature must also be
  reachable through the Store API.

## 2. Pages and pagelets

A "page" bundles a controller, a **page loader**, a **page struct**, and a
**page-loaded event**. The page loader:

- Takes `GenericPageLoaderInterface` (loads default meta data) and
  `EventDispatcherInterface` in its constructor.
- Has a `load(Request, SalesChannelContext)` method (by convention) that calls
  `$this->genericPageLoader->load(...)`, then `MyPage::createFrom($page)`
  (`Page` extends `Struct`, providing `createFrom`), fills the page via setters,
  dispatches a `*PageLoadedEvent`, and returns the page.
- **Must not** use the DAL/repository directly — load data from a Store API
  route instead.

The page struct extends `Shopware\Storefront\Page\Page` and exposes
getters/setters for custom data. The event extends
`Shopware\Storefront\Page\PageLoadedEvent`, holds the page, and forwards
`SalesChannelContext` + `Request` to its parent constructor. The controller
injects the loader, calls `load()`, and passes the page into `renderStorefront`.

Pagelets are reusable fractions of a page following the same pattern. Full
class set in [references/pages.md](references/pages.md).

## 3. Adding data to an existing page

To add data without owning the page, subscribe to its `*LoadedEvent` and attach
an extension:

1. Implement `EventSubscriberInterface`, subscribe to e.g.
   `FooterPageletLoadedEvent` (from
   `Shopware\Storefront\Pagelet\Footer`).
2. Fetch data through a Store API route (not the DAL directly — pages/pagelets
   must not call the repository in the subscriber for this kind of data).
3. Attach it: `$event->getPagelet()->addExtension('product_count', $result);`.
4. Register the subscriber with `->tag('kernel.event_subscriber')`.
5. Read it in an overridden template via `footer.extensions.product_count`.

Worked example with a custom Store API route in
[references/add-data-to-page.md](references/add-data-to-page.md).

## 4. Customizing templates

Shopware loads plugin templates from `<plugin root>/src/Resources/views`.
To override a core template, recreate the **exact same path** starting from
`views` (e.g. core
`storefront/layout/header/logo.html.twig` → your
`Resources/views/storefront/layout/header/logo.html.twig`).

In your file:

```twig
{% sw_extends '@Storefront/storefront/layout/header/logo.html.twig' %}

{% block layout_header_logo_link %}
    <h2>Hello world!</h2>
{% endblock %}
```

- `{% sw_extends %}` inherits the original; redefine only the blocks you change.
- Use `{{ parent() }}` to append to the original block content instead of
  replacing it.
- `{{ dump() }}` lists all variables available on the page.
- Clear the cache after changes (`bin/console cache:clear`) and assign your
  theme to the sales channel.

## 5. Custom Twig function

Register a Twig extension to expose a PHP function in templates:

```php
namespace SwagBasicExample\Twig;

use Twig\Extension\AbstractExtension;
use Twig\TwigFunction;

class SwagCreateMd5Hash extends AbstractExtension
{
    public function getFunctions()
    {
        return [new TwigFunction('createMd5Hash', [$this, 'createMd5Hash'])];
    }

    public function createMd5Hash(string $str)
    {
        return md5($str);
    }
}
```

Register it in `services.php` as `public()` with `->tag('twig.extension')`
(required). The function is then callable in any template:
`{% set md5Hash = createMd5Hash('Shopware is awesome') %}`. Do **not** use Twig
functions to fetch database data — use a `DataResolver` for that.

## References

- [references/controllers.md](references/controllers.md) — full controller + DI + routing + template wiring
- [references/pages.md](references/pages.md) — page loader, page struct, page-loaded event
- [references/add-data-to-page.md](references/add-data-to-page.md) — subscriber + Store API route to extend a page
