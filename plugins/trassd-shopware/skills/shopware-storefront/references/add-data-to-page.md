# Adding data to an existing Storefront page

When you don't own the page, listen to its `*LoadedEvent`, fetch the data
through a Store API route, attach it as an extension, and read it in an
overridden template. Example: show the active product count in the footer.

Workflow:

1. Find the page/pagelet to extend.
2. Subscribe to its `Loaded` event.
3. Add a Store API route for the data you need.
4. Attach the data to the page/pagelet via the event.
5. Display it in an overridden template.

## Subscriber

```php
// PLUGIN_ROOT/src/Service/AddDataToPage.php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Service;

use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Storefront\Pagelet\Footer\FooterPageletLoadedEvent;
use Swag\BasicExample\Core\Content\Example\SalesChannel\ProductCountRoute;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class AddDataToPage implements EventSubscriberInterface
{
    public function __construct(private ProductCountRoute $productCountRoute)
    {
    }

    public static function getSubscribedEvents(): array
    {
        return [
            FooterPageletLoadedEvent::class => 'addActiveProductCount',
        ];
    }

    public function addActiveProductCount(FooterPageletLoadedEvent $event): void
    {
        $response = $this->productCountRoute->load(new Criteria(), $event->getSalesChannelContext());

        $event->getPagelet()->addExtension('product_count', $response->getProductCount());
    }
}
```

`addExtension($name, $data)` makes the value reachable in the template at
`<object>.extensions.<name>`. Inside a pagelet event, do not call the DAL
directly — go through a Store API route.

## Store API route

A custom route returns exactly the aggregated data needed (here a count),
rather than reusing a broad route like `ProductListRoute`. Define an abstract
base (`StoreApiRouteScope::ID`), the concrete route, and a response struct
extending `StoreApiResponse`; the route uses `aggregate()` with a
`CountAggregation` rather than `search()`. Register the route service and
import `../../Core/**/*Route.php` (attribute) in `routes.php`. See the
`shopware-store-api` material for the full route/response pattern.

## Service registration

```php
// PLUGIN_ROOT/src/Resources/config/services.php
$services->set(ProductCountRoute::class)
    ->public()
    ->args([service('product.repository')]);

$services->set(AddDataToPage::class)
    ->args([service(ProductCountRoute::class)])
    ->tag('kernel.event_subscriber');
```

## Template

Override the footer template and render the extension:

```twig
{# PLUGIN_ROOT/src/Resources/views/storefront/layout/footer/footer.html.twig #}
{% sw_extends '@Storefront/storefront/layout/footer/footer.html.twig' %}

{% block layout_footer_navigation_columns %}
    {{ parent() }}

    {% if footer.extensions.product_count %}
        <div class="col-md-4 footer-column">
            <p>This shop offers you {{ footer.extensions.product_count.count }} products</p>
        </div>
    {% endif %}
{% endblock %}
```

The data lives under `footer.extensions.product_count` — the name passed to
`addExtension`.
