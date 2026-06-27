---
name: contao-page-controllers
description: >-
  Implement custom Contao page types and back end routes with controllers — the
  #[AsPage] attribute / page controllers (type, tl_page DCA registration,
  __invoke response, content composition) and registering custom back end
  routes (the Route, _scope: backend, AbstractBackendController). Triggers when
  adding a custom page type or a back end route in a Contao bundle.
---

# Contao Page Controllers & Back End Routes

Contao is a Symfony-based CMS. Two related patterns let a bundle handle requests
with full routing control:

- **Page controllers** — a page type in Contao's site structure backed by a
  controller, so an editor defines the route (alias) in the back end while you
  keep control over routing and the response.
- **Back end routes** — a custom controller rendered inside the Contao back end
  (e.g. a custom admin screen) without DCA.

## Page controllers

### Register with `#[AsPage]`

A page controller is an invokable class tagged for Contao's page routing. Use
the `Contao\CoreBundle\DependencyInjection\Attribute\AsPage` attribute (this is
the modern equivalent of the `contao.page` service tag / `@Page` annotation):

```php
// src/Controller/Page/ExamplePageController.php
namespace App\Controller\Page;

use Contao\CoreBundle\DependencyInjection\Attribute\AsPage;
use Symfony\Component\HttpFoundation\Response;

#[AsPage]
class ExamplePageController
{
    public function __invoke(): Response
    {
        return new Response('Hello World!');
    }
}
```

The page **type** is inferred from the class name when not given — the
`Page`/`Controller` suffixes are stripped, so this class registers the type
`example`. Set it explicitly with `#[AsPage(type: 'custom_type')]`.

> `type` is the **first** `AsPage` argument — unlike Symfony's `#[Route]`, where
> the first argument is the path.

### Register the page type in the back end

A page type also needs a `tl_page` palette and a label, or it cannot be added in
the site structure:

```php
// contao/dca/tl_page.php
$GLOBALS['TL_DCA']['tl_page']['palettes']['example'] =
    '{title_legend},title,type,alias;{publish_legend},published,start,stop';
```

```php
// contao/languages/en/default.php
$GLOBALS['TL_LANG']['PTY']['example'] = ['Example', 'Example page type.'];
```

The page's **alias** (defined in the back end) becomes its route.

### `AsPage` parameters

`AsPage` accepts the usual Symfony routing options (`requirements`, `options`,
`methods`, `defaults`) plus Contao-specific ones. Key parameters:

- **`path`** — controls the URL relative to the alias.
  - Absolute (`/foo/bar`): URL is always `/foo/bar` + suffix, regardless of the
    alias — so do **not** add the `alias` field to the palette.
  - Relative (`foo/bar`): appended to the alias (`alias/foo/bar`).
  - May contain parameters (`{foobar}`), optional via `defaults`. Never name a
    parameter `parameters` — it is reserved by Contao's page routing.
- **`urlSuffix`** — overrides the root page's URL suffix for this type
  (e.g. `.xml`, `.csv`), enabling feed-style pages.
- **`contentComposition`** (default `true`) — whether back-end articles/content
  can be assigned. Set `false` for pages whose body is fully controlled by the
  controller (e.g. an RSS feed).

```php
#[AsPage(type: 'example', path: '/foo/bar', urlSuffix: '.html', contentComposition: true)]
```

### Routable vs. content-composition pages

- A **simple/routable** page returns its own `Response` from `__invoke()` and is
  typically declared `contentComposition: false`.
- A **content-composition** page should render the configured page layout.
  Extend `Contao\CoreBundle\Controller\Page\AbstractPageController` and call its
  protected `renderPage(PageModel $pageModel)` to render the layout (the base
  class is the modern, service-aware alternative to manually using the legacy
  `FrontendIndex`).

### Injecting the `PageModel` and generating URLs

Contao extends Symfony's argument value resolver, so you can type-hint the
current page's `Contao\PageModel` (and the `Request`) as controller arguments:

```php
public function __invoke(Request $request, PageModel $pageModel): Response
{
    return new Response('Hello page: '.$pageModel->title);
}
```

Generate URLs to pages via `PageModel::getFrontendUrl()` /
`getAbsoluteUrl()`. For controllers whose `path` has **mandatory parameters**,
those simple calls fail; instead generate via Symfony's `UrlGeneratorInterface`
using the shared page route name
`PageRoute::PAGE_BASED_ROUTE_NAME` and pass the page as the route's content
object. See [references/url-generation.md](references/url-generation.md).

## Back end routes

To render your own controller inside the Contao back end, extend
`Contao\CoreBundle\Controller\AbstractBackendController` and define a normal
Symfony route. Two essentials:

- Prefix the path with the parameter `%contao.backend.route_prefix%` so the
  route lives under the back end path.
- Add the request attribute `'_scope' => 'backend'` via route `defaults` — this
  tells Contao the route belongs to the back end and must be handled in the
  back end scope.

```php
// src/Controller/BackendController.php
namespace App\Controller;

use Contao\CoreBundle\Controller\AbstractBackendController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

#[Route('%contao.backend.route_prefix%/my-backend-route', name: self::class, defaults: ['_scope' => 'backend'])]
class BackendController extends AbstractBackendController
{
    public function __invoke(): Response
    {
        return $this->render('my_backend_route.html.twig', [
            'title' => 'My title',
            'headline' => 'My headline',
            'foo' => 'bar',
        ]);
    }
}
```

Import the controllers and extend the back end Twig base template
`@Contao/be_main`:

```yaml
# config/routes.yaml
app.controller:
    resource: ../src/Controller
    type: attribute
```

```twig
{% extends "@Contao/be_main" %}
{% block main_content %}
    <div class="tl_listing_container">Main Content: {{ foo }}</div>
{% endblock %}
```

Load these routes **before** the `ContaoCoreBundle` routes. In a Managed Edition
bundle, register them via the `RoutingPluginInterface` in your `Plugin` class.

### Adding a back end menu entry

A controller is reachable by URL but not yet in the menu. Add a menu node from
an event listener on the `contao.backend_menu_build` event. See
[references/back-end-menu.md](references/back-end-menu.md).

## Checklist

- [ ] Page controller is invokable and tagged with `#[AsPage]` (or
      `contao.page`); `type` set or correctly inferred from the class name.
- [ ] `tl_page` palette **and** `TL_LANG['PTY']` label exist for the type.
- [ ] `alias` is omitted from the palette only when `path` is absolute.
- [ ] No route parameter is named `parameters`.
- [ ] Feed/non-content pages set `contentComposition: false`; layout pages
      extend `AbstractPageController` and call `renderPage()`.
- [ ] Back end route has `_scope: backend` and the
      `%contao.backend.route_prefix%` prefix, extends
      `AbstractBackendController`, and renders an `@Contao/be_main` template.
- [ ] Bundle routes are loaded before `ContaoCoreBundle`'s.
