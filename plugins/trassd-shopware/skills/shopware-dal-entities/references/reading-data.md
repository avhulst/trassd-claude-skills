# Reading data (EntityRepository + Criteria)

Shopware uses no ORM — read through the auto-generated repository
(`<entity>.repository`) with a `Criteria` object.

## Inject the repository

```php
// services.php
use function Symfony\Component\DependencyInjection\Loader\Configurator\service;

$services->set(ReadingData::class)
    ->args([service('product.repository')]);
```

```php
use Shopware\Core\Framework\DataAbstractionLayer\EntityRepository;

class ReadingData
{
    public function __construct(private EntityRepository $productRepository) {}
}
```

## Basic search

`search()` returns an `EntitySearchResult` (extends the iterable
`EntityCollection`); `first()` returns one entity or `null`.

```php
use Shopware\Core\Framework\Context;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;

$all   = $this->productRepository->search(new Criteria(), $context);
$byId  = $this->productRepository->search(new Criteria([$id]), $context)->first();
```

## Filters

`Shopware\Core\Framework\DataAbstractionLayer\Search\Filter\*`.

```php
use Shopware\Core\Framework\DataAbstractionLayer\Search\Filter\EqualsFilter;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Filter\OrFilter;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Filter\RangeFilter;

$criteria = new Criteria();
$criteria->addFilter(new EqualsFilter('name', 'Example name'));      // WHERE name = ...

$criteria->addFilter(new OrFilter([
    new EqualsFilter('id', $id),
    new EqualsFilter('name', 'Example name'),
]));

$criteria->addFilter(new RangeFilter('points', [RangeFilter::GTE => 4]));
```

Combine with `AndFilter` / `OrFilter` / `NandFilter`. Use `addPostFilter()` to
filter the result **without** affecting aggregations.

## Associations

```php
$criteria->addAssociation('productReviews');           // load association
$criteria->addAssociation('productReviews.customer');  // chained / nested

// filter the association itself (returns a nested Criteria):
$criteria->getAssociation('productReviews')
    ->addFilter(new RangeFilter('points', [RangeFilter::GTE => 4]));
```

`addAssociation()` loads matching related rows but still returns the parent even
if none match. Filtering on the dotted path
(`addFilter(new RangeFilter('productReviews.points', ...))`) restricts the parent
result to those that have a matching association.

## Aggregations

```php
use Shopware\Core\Framework\DataAbstractionLayer\Search\Aggregation\Metric\AvgAggregation;

$criteria->addAggregation(new AvgAggregation('avg-rating', 'productReviews.points'));
$result = $this->productRepository->search($criteria, $context); // no first()
$rating = $result->getAggregations()->get('avg-rating');
```

## Limiting, paging, sorting

```php
use Shopware\Core\Framework\DataAbstractionLayer\Search\Sorting\FieldSorting;

$criteria->setLimit(10);
$criteria->setOffset(0);
$criteria->addSorting(new FieldSorting('createdAt', FieldSorting::ASCENDING));
```

## Mapping (ManyToMany) entities

Cannot be read with `search()` — they are only two primary keys. Use:

```php
$ids = $this->productCategoryRepository->searchIds($criteria, $context);
```

## Large datasets — RepositoryIterator

Fetches in batches to avoid memory exhaustion. Sort by a **deterministic** key
(add `FieldSorting('id')` when the primary sort isn't unique) so rows aren't
duplicated or skipped across batches.

```php
use Shopware\Core\Framework\DataAbstractionLayer\Dbal\Common\RepositoryIterator;

$criteria = new Criteria();
$criteria->addSorting(new FieldSorting('id'));
$criteria->setLimit(500);

$iterator = new RepositoryIterator($this->productRepository, $context, $criteria);
while (($result = $iterator->fetch()) !== null) {
    foreach ($result->getEntities() as $product) {
        // ...
    }
}
```

All available fields/associations for an entity live in its definition (e.g.
`ProductDefinition`). Full filter and aggregation lists are in Shopware's DAL
reference.
