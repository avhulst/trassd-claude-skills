# Custom page — loader, struct, event

A page consists of a controller, a page loader, a page struct, and a
page-loaded event. The page struct (and pagelets) are the objects handed to the
template.

## Page loader

The loader builds the page and dispatches its loaded event. It does **not**
extend any base class. It must not query the DAL directly — fetch extra data
from a Store API route.

```php
// PLUGIN_ROOT/src/Storefront/Page/Example/ExamplePageLoader.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Storefront\Page\Example;

use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Shopware\Storefront\Page\GenericPageLoaderInterface;
use Symfony\Component\EventDispatcher\EventDispatcherInterface;
use Symfony\Component\HttpFoundation\Request;

class ExamplePageLoader
{
    public function __construct(
        private GenericPageLoaderInterface $genericPageLoader,
        private EventDispatcherInterface $eventDispatcher
    ) {
    }

    public function load(Request $request, SalesChannelContext $context): ExamplePage
    {
        $page = $this->genericPageLoader->load($request, $context);
        $page = ExamplePage::createFrom($page);

        // load extra data from a store-api route, then set it
        $page->setExampleData(/* ... */);

        $this->eventDispatcher->dispatch(
            new ExamplePageLoadedEvent($page, $context, $request)
        );

        return $page;
    }
}
```

`GenericPageLoaderInterface` provides the default page meta-data.
`ExamplePage::createFrom($page)` works because `Page` extends `Struct` (which
supplies the `createFrom` helper). The convention is a `load()` method so the
loader behaves like core loaders and can be decorated (optionally back it with
an interface).

Register it:

```php
// PLUGIN_ROOT/src/Resources/config/services.php
$services->set(ExamplePageLoader::class)
    ->public()
    ->args([
        service('Shopware\Storefront\Page\GenericPageLoader'),
        service('event_dispatcher'),
    ]);
```

## Page struct

Extends `Shopware\Storefront\Page\Page`; carries the custom data with a
getter/setter pair.

```php
// PLUGIN_ROOT/src/Storefront/Page/Example/ExamplePage.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Storefront\Page\Example;

use Shopware\Storefront\Page\Page;
use Swag\BasicExample\Core\Content\Example\ExampleEntity;

class ExamplePage extends Page
{
    protected ExampleEntity $exampleData;

    public function getExampleData(): ExampleEntity
    {
        return $this->exampleData;
    }

    public function setExampleData(ExampleEntity $exampleData): void
    {
        $this->exampleData = $exampleData;
    }
}
```

## Page-loaded event

Extends `Shopware\Storefront\Page\PageLoadedEvent`; holds the page and forwards
context + request to the parent.

```php
// PLUGIN_ROOT/src/Storefront/Page/Example/ExamplePageLoadedEvent.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Storefront\Page\Example;

use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Shopware\Storefront\Page\PageLoadedEvent;
use Symfony\Component\HttpFoundation\Request;

class ExamplePageLoadedEvent extends PageLoadedEvent
{
    protected ExamplePage $page;

    public function __construct(ExamplePage $page, SalesChannelContext $salesChannelContext, Request $request)
    {
        $this->page = $page;
        parent::__construct($salesChannelContext, $request);
    }

    public function getPage(): ExamplePage
    {
        return $this->page;
    }
}
```

## Controller using the loader

Inject the loader, call `load()`, hand the page to the template:

```php
class ExampleController extends StorefrontController
{
    public function __construct(private ExamplePageLoader $examplePageLoader)
    {
    }

    #[Route(path: '/example-page', name: 'frontend.example.page', methods: ['GET'])]
    public function examplePage(Request $request, SalesChannelContext $context): Response
    {
        $page = $this->examplePageLoader->load($request, $context);

        return $this->renderStorefront('@SwagBasicExample/storefront/page/example/index.html.twig', [
            'page' => $page,
        ]);
    }
}
```

Pass the loader into the controller service via `->args([service(ExamplePageLoader::class)])`.

Pagelets ("reusable fractions of a page") follow the same loader/struct/event
pattern.
