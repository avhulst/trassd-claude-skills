# Cache tagging and invalidation — examples

All examples are grounded in `framework/caching.md`.

## Add tags from a plain controller

Inject the response tagger service `fos_http_cache.http.symfony_response_tagger`
(`FOS\HttpCache\ResponseTagger`). Tags are collected during the request and
attached to the response on the `kernel.response` event.

```php
// src/Controller/MySuperController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use FOS\HttpCache\ResponseTagger;

class MySuperController
{
    public function __construct(private ResponseTagger $responseTagger)
    {
    }

    public function __invoke(): Response
    {
        $this->responseTagger->addTags(['news-42']);

        return new Response();
    }
}
```

## Add tags from a fragment controller

Extend `Contao\CoreBundle\Controller\AbstractFragmentController` (here via the
content-element base class) and use `tagResponse()` for convenience:

```php
// src/Controller/ContentElement/MyContentElementController.php
namespace App\Controller\ContentElement;

use Contao\ContentModel;
use Contao\CoreBundle\Controller\ContentElement\AbstractContentElementController;
use Contao\CoreBundle\DependencyInjection\Attribute\AsContentElement;
use Contao\CoreBundle\Twig\FragmentTemplate;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

#[AsContentElement(category: 'texts')]
class MyContentElementController extends AbstractContentElementController
{
    protected function getResponse(FragmentTemplate $template, ContentModel $model, Request $request): Response
    {
        // ...

        $this->tagResponse(['news-42']);

        return $template->getResponse();
    }
}
```

## Invalidate tags

Inject `fos_http_cache.cache_manager` (`FOS\HttpCacheBundle\CacheManager`).
Invalidation requests are collected and executed on `kernel.terminate`.

```php
// src/EventListener/UserChangedSomethingListener.php
namespace App\EventListener;

use FOS\HttpCacheBundle\CacheManager;

class UserChangedSomethingListener
{
    public function __construct(private CacheManager $cacheManager)
    {
    }

    public function onUserChangedSomething(): void
    {
        $this->cacheManager->invalidateTags(['news-42']);
    }
}
```

## Back end auto-invalidation

Editing a record automatically invalidates a conventional tag set. Example —
editing news article ID 42 (parent `tl_news_archive` ID 1, children
`tl_content` 420 and 421):

- `contao.db.tl_news_archive`    (topmost parent table)
- `contao.db.tl_news_archive.1`  (parent record)
- `contao.db.tl_news.42`         (the record itself)
- `contao.db.tl_content.420`     (first child record)
- `contao.db.tl_content.421`     (second child record)

Only the **topmost** parent *table* tag is invalidated (here
`contao.db.tl_news_archive`), not `contao.db.tl_news` or `contao.db.tl_content`.

For a custom DCA `tl_contact_details` (no parent/child) editing ID 42
invalidates `contao.db.tl_contact_details.42` and `contao.db.tl_contact_details`.

To add your own tags during this process, register the
`oninvalidate_cache_tags` DCA callback.

## Tag by entity / model class

The `Contao\CoreBundle\Cache\CacheTagManager` service (the doc's "EntityCacheTags
helper") tags and invalidates based on entity or model classes/instances, e.g.
`tagWithModelInstance($model)`, `invalidateTagsForModelClass(News::class)`.
