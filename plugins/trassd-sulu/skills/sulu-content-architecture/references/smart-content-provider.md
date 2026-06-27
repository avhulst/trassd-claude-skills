# Custom SmartContentProvider — full example

A `SmartContentProvider` loads the data for a `smart_content` property. It
resolves the editor-configured filter array and returns matching items. The
example below builds a provider for an `Example` entity using the configuration
`Builder`.

## Filter configuration the provider receives

The backend overlay produces a filter array with values such as:

| Name | Meaning |
| --- | --- |
| `dataSource` | Additional constraint (e.g. a page "folder"). |
| `tags` (`int[]`) + `tagOperator` | Selected tag IDs; any/all. |
| `categories` + `categoryOperator` | Selected categories; any/all. |
| `types` | Selected types (e.g. templates). |

`websiteTags` / `websiteCategories` (with their own operators) can be injected
via GET params from the website, separate from admin-selected values.
Pagination (`page` & `pageSize`) and `limit` are also available, plus
`presentAs` for website display options.

## 1. Repository (optional)

Query logic may live in a repository or directly in the provider's `findFlatBy`
/ `countBy`.

```php
<?php

declare(strict_types=1);

namespace App\Repository;

use Doctrine\ORM\EntityRepository;

class ExampleRepository extends EntityRepository
{
    public function findByFilters(array $filters, array $sortBys, array $params = []): array
    {
        $qb = $this->createQueryBuilder('example');

        if (!empty($filters['tags'])) {
            $qb->join('example.tags', 'tag')
                ->andWhere('tag.id IN (:tags)')
                ->setParameter('tags', $filters['tags']);
        }

        foreach ($sortBys as $sortBy) {
            $qb->addOrderBy('example.' . $sortBy['column'], $sortBy['direction']);
        }

        if (isset($params['limit'])) {
            $qb->setMaxResults($params['limit']);
        }
        if (isset($params['offset'])) {
            $qb->setFirstResult($params['offset']);
        }

        return $qb->getQuery()->getResult();
    }

    public function countByFilters(array $filters): int
    {
        $qb = $this->createQueryBuilder('example')->select('COUNT(example.id)');

        if (!empty($filters['tags'])) {
            $qb->join('example.tags', 'tag')
                ->andWhere('tag.id IN (:tags)')
                ->setParameter('tags', $filters['tags']);
        }

        return (int) $qb->getQuery()->getSingleScalarResult();
    }
}
```

## 2. Provider

```php
<?php

declare(strict_types=1);

namespace App\SmartContent;

use App\Entity\Example;
use App\Repository\ExampleRepository;
use App\ResourceLoader\ExampleResourceLoader;
use Sulu\Bundle\AdminBundle\SmartContent\Configuration\Builder;
use Sulu\Bundle\AdminBundle\SmartContent\Configuration\ProviderConfigurationInterface;
use Sulu\Bundle\AdminBundle\SmartContent\SmartContentProviderInterface;

class ExampleSmartContentProvider implements SmartContentProviderInterface
{
    public function __construct(private ExampleRepository $repository)
    {
    }

    public function getConfiguration(): ProviderConfigurationInterface
    {
        return Builder::create()
            ->enableTags()
            ->enableCategories()
            ->enableLimit()
            ->enablePagination()
            ->enableSorting([
                ['column' => 'created', 'title' => 'sulu_admin.created'],
                ['column' => 'title', 'title' => 'sulu_admin.title'],
            ])
            ->enableTypes([
                ['type' => 'example-type-1', 'title' => 'app.example_type_1'],
            ])
            ->enableView('app.example_edit_form', ['id' => 'id'])
            ->getConfiguration();
    }

    public function countBy(array $filters, array $params = []): int
    {
        return $this->repository->countByFilters($filters);
    }

    /** @return array<array{id: string, title: string}> */
    public function findFlatBy(array $filters, array $sortBys, array $params = []): array
    {
        $entities = $this->repository->findByFilters($filters, $sortBys, $params);

        return array_map(
            static fn (Example $e) => ['id' => (string) $e->getId(), 'title' => $e->getTitle()],
            $entities,
        );
    }

    public function getType(): string
    {
        return 'examples';
    }

    public function getResourceLoaderKey(): string
    {
        return ExampleResourceLoader::RESOURCE_LOADER_KEY;
    }
}
```

`findFlatBy` returns only `id` + `title`. Sulu uses the IDs to decide which
records to render and delegates loading to the `ResourceLoader` named by
`getResourceLoaderKey()` (a service tagged `sulu_content.resource_loader`
implementing
`Sulu\Content\Application\ResourceLoader\Loader\ResourceLoaderInterface`).

## Builder methods

| Method | Effect |
| --- | --- |
| `enableTags()` / `enableCategories()` | Tag / category filtering. |
| `enableLimit()` | Limit output count. |
| `enablePagination()` | Pagination + items per page. |
| `enablePresentAs()` | Multiple view options (configured in the template). |
| `enableSorting(array $sorting)` | Sorting; pass the options. |
| `enableTypes(array $types)` | Type filtering; pass selectable types. |
| `enableDatasource(string $resourceKey, string $listKey, string $adapter)` | Source selection (tree structures, e.g. pages below a parent). |
| `enableAudienceTargeting()` | Audience-targeting filtering. |
| `enableProperties(array $properties)` | Default property names for HTML templates (only for ContentRichEntities). |
| `enableView(string $view, array $resultToView)` | Target view when clicking a result; maps clicked-item values to the view path via a JSON pointer. |

## 3. Service definition

```yaml
# config/services.yaml
services:
    App\SmartContent\ExampleSmartContentProvider:
        arguments:
            - '@App\Repository\ExampleRepository'
        tags:
            - { name: 'sulu_content.smart_content_provider', type: 'examples' }
```

The `type` attribute must equal `getType()`. Then set `provider: examples` on
the `smart_content` property in the template to use it.
