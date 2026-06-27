# PropertyResolver & ResourceLoader — full example

A **PropertyResolver** turns raw property data into a `ContentView` of
`ResolvableResource` placeholders; a **ResourceLoader** batch-loads the actual
resources by ID. Together they implement a custom selection field (here:
selecting `Product` entities on a page).

See the resolution pipeline in `SKILL.md` ("How content is resolved for the
frontend") for how these two are invoked.

## ResourceLoader

Implements
`Sulu\Content\Application\ResourceLoader\Loader\ResourceLoaderInterface`
(`load(array $ids, ?string $locale, array $params = []): array`) plus a
`getKey()` method used to index the loader.

```php
<?php

declare(strict_types=1);

namespace App\Content\ResourceLoader;

use App\Repository\ProductRepository;
use Sulu\Content\Application\ResourceLoader\Loader\ResourceLoaderInterface;

class ProductResourceLoader implements ResourceLoaderInterface
{
    public const RESOURCE_LOADER_KEY = 'products';

    public function __construct(private ProductRepository $productRepository)
    {
    }

    public function load(array $ids, ?string $locale, array $params = []): array
    {
        $filters = ['ids' => $ids, 'locale' => $locale, 'published' => true];

        if (isset($params['filters']) && is_array($params['filters'])) {
            $filters = array_merge($filters, $params['filters']);
        }

        $mappedResult = [];
        foreach ($this->productRepository->findByFilters($filters) as $product) {
            $mappedResult[$product->getId()] = $product; // index by ID — required
        }

        return $mappedResult;
    }

    public static function getKey(): string
    {
        return self::RESOURCE_LOADER_KEY;
    }
}
```

Rules: always batch-load in a single query (no N+1); return results indexed by
ID; respect `$locale`; allow `params` to override filters; omit missing
resources (handled gracefully); put security/permission checks here.

## PropertyResolver

Implements
`Sulu\Content\Application\PropertyResolver\Resolver\PropertyResolverInterface`
(`resolve(mixed $data, string $locale, array $params = []): ContentView`) plus a
`getType()` method used to index it by property type.

```php
<?php

declare(strict_types=1);

namespace App\Content\PropertyResolver;

use App\Entity\Product;
use App\Content\ResourceLoader\ProductResourceLoader;
use Sulu\Content\Application\ContentResolver\Value\ContentView;
use Sulu\Content\Application\PropertyResolver\Resolver\PropertyResolverInterface;

class ProductSelectionPropertyResolver implements PropertyResolverInterface
{
    public function resolve(mixed $data, string $locale, array $params = []): ContentView
    {
        if (!is_array($data) || 0 === count($data) || !array_is_list($data)) {
            return ContentView::create([], ['ids' => [], ...$params]);
        }

        $ids = $data;
        $resourceLoaderKey = $params['resourceLoader'] ?? ProductResourceLoader::getKey();

        return ContentView::createResolvablesWithReferences(
            ids: $ids,
            resourceLoaderKey: $resourceLoaderKey,
            resourceKey: Product::RESOURCE_KEY,
            view: ['ids' => $ids, ...$params],
            priority: 150,
            metadata: ['properties' => $params['properties'] ?? null],
        );
    }

    public static function getType(): string
    {
        return 'product_selection';
    }
}
```

Key points:

- Validate input; return an empty `ContentView` for invalid data — never throw.
- Factory choice: `createResolvablesWithReferences()` (entity selections —
  enables cache invalidation), `createResolvable()` (single resource),
  `create()` (simple data, no loading).
- Priority convention: `-50` links/media, `0` default, `100` articles/snippets,
  `150` pages (higher reserved, e.g. `2048` for `SmartResolvable`). Same
  priority + loader key = batched together.
- `resourceKey` (e.g. `Product::RESOURCE_KEY`) is used for reference tracking
  and cache invalidation.
- `metadata` controls which properties are resolved for nested content
  entities.

## Service definitions

With autowiring (Sulu default), the tags are applied automatically to all
implementers of the interfaces — manual tagging is optional.

```yaml
# config/services.yaml
services:
    App\Content\ResourceLoader\ProductResourceLoader:
        tags: [{ name: 'sulu_content.resource_loader' }]

    App\Content\PropertyResolver\ProductSelectionPropertyResolver:
        tags: [{ name: 'sulu_content.property_resolver' }]
```

The `sulu_content.property_resolver` tag indexes by `getType()`; the
`sulu_content.resource_loader` tag indexes by `getKey()`.

## Template + Twig

```xml
<property name="products" type="product_selection">
    <meta><title lang="en">Featured Products</title></meta>
</property>
```

```twig
{% for product in content.products %}
    <h3>{{ product.title }}</h3>
{% endfor %}
{% if content.products is empty %}<p>No products selected.</p>{% endif %}
```

## Best practices & pitfalls

- Never load resources in a PropertyResolver — that is the ResourceLoader's job.
- Always index ResourceLoader results by ID.
- These services are read-only frontend services. For admin/write use Sulu's
  Admin API and form metadata system.
