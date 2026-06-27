# Caching fragments — examples

All examples are grounded in `framework/caching.md`.

Content elements and front end modules are fragment controllers. Each returns a
`Response` and therefore declares whether (and how long) it can be cached.

## Inline fragments (default)

Rendered inside the main page request and merged before caching. The page's
cache time and the fragment's cache time merge to the **lowest common
denominator**: if the page is cacheable for a day but the fragment only for an
hour, the whole page is cached for one hour.

A fragment can also *raise* effective expiry only down to its own limit. A
"current year" fragment is cacheable until the end of 31 December; cached on
10 December for a day the page time is unaffected, but cached at 12:00 on
31 December it drops to 12 hours so the cache expires at year end.

```php
class CurrentYearController extends AbstractFrontendModuleController
{
    protected function getResponse(FragmentTemplate $template, ModuleModel $model, Request $request): Response
    {
        $year = (int) date('Y');
        $template->set('year', $year);
        $response = $template->getResponse();

        // Cache until the end of the year
        $response->setPublic();
        $response->setMaxAge(strtotime(($year + 1).'-01-01 00:00:00') - time());

        return $response;
    }
}
```

### How merging works under the hood

A Symfony `Response` is `Cache-Control: private` by default, so fragments are
not merged unless they carry a marker header:

```php
$response->headers->set('Contao-Merge-Cache-Control', true);
```

This is applied automatically for responses created from a Contao template, so
prefer `$template->getResponse()` and just set the cache directives:

```php
// src/Controller/MySuperController.php
namespace App\Controller;

use Contao\CoreBundle\Controller\FrontendModule\AbstractFrontendModuleController;
use Contao\CoreBundle\Twig\FragmentTemplate;
use Contao\ModuleModel;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

class MySuperController extends AbstractFrontendModuleController
{
    protected function getResponse(FragmentTemplate $template, ModuleModel $model, Request $request): Response
    {
        $response = $template->getResponse();

        $response->setPublic();
        $response->setMaxAge(3600);

        return $response;
    }
}
```

## ESI (Edge Side Includes)

Set the fragment renderer to `esi` to cache the fragment **separately** from the
main content. The page may be cached for 24 hours while the fragment is cached
for a week; on each request the reverse proxy merges them, rebuilding only the
expired piece.

Use ESI only when the fragment is **cacheable** and **expensive** to generate
(e.g. a weather preview backed by a once-a-day API call). Warnings from the doc:

- If the fragment is `private` or uncacheable, ESI forces the proxy to boot the
  whole system on every request to render it — no benefit, possibly slower.
- Multiple ESI fragments each fetched individually can be slower than rendering
  everything together. For cheap fragments (e.g. `{{date::Y}}`), cache the whole
  page and regenerate more often instead.
- Many cases are better solved client-side with JavaScript.

Symfony auto-detects whether the gateway cache supports ESI (Symfony's built-in
proxy used by the Managed Edition does; Varnish and several CDNs also do). If
not supported, fragments render **inline** automatically. There is also an
`hinclude` renderer for client-side merging (client part not shipped).
