# Generating URLs for page controllers

Pages live in the `tl_page` table and are represented by `Contao\PageModel`.
The model can build URLs to itself:

- `getFrontendUrl()` — path-absolute URL (relative to the document, unless the
  page is on another domain).
- `getAbsoluteUrl()` — always absolute, including the scheme.

Both accept optional **path parameters**. As a string they map to legacy
`auto_item` / path parameters:

```php
$page->getAbsoluteUrl();          // https://example.com/alias-of-the-page.html
$page->getAbsoluteUrl('/foobar'); // https://example.com/alias-of-the-page/foobar.html
```

## Caveat: mandatory route parameters

If a page controller declares a path with mandatory parameters, e.g.

```php
#[AsPage(path: '{foo}/{bar}')]
```

then `$page->getFrontendUrl()` fails, because `foo` and `bar` are unknown.

Generate the URL through Symfony's `UrlGeneratorInterface`. The route name is
**not** the page type — it is the shared page route
`PageRoute::PAGE_BASED_ROUTE_NAME` (`page_routing_object`). Pass the page model
as the route's content object plus your real parameters:

```php
use Contao\PageModel;
use Contao\CoreBundle\Routing\Page\PageRoute;
use Symfony\Cmf\Component\Routing\RouteObjectInterface;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

class MyService
{
    public function __construct(private readonly UrlGeneratorInterface $urlGenerator)
    {
    }

    private function getUrlForPage(PageModel $page): string
    {
        return $this->urlGenerator->generate(
            PageRoute::PAGE_BASED_ROUTE_NAME,
            [
                RouteObjectInterface::CONTENT_OBJECT => $page,
                'foo' => 'lorem',
                'bar' => 'ipsum',
            ],
        );
    }
}
```

## Contao 5.3+

From Contao 5.3 you can instead pass the parameters as an array directly to the
model methods:

```php
$page->getFrontendUrl([
    'foo' => 'lorem',
    'bar' => 'ipsum',
]);
```
