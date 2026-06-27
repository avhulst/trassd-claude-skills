# Repository query patterns

Custom repositories extend `ServiceEntityRepository` and are autowirable
services. Choose the query style by need: QueryBuilder for dynamic queries, DQL
for static ones, raw SQL only when necessary.

## DQL via `createQuery()`

Doctrine Query Language reads like SQL but references entity classes and
properties, returning hydrated objects.

```php
// src/Repository/ProductRepository.php
class ProductRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Product::class);
    }

    /** @return Product[] */
    public function findAllGreaterThanPrice(int $price): array
    {
        $query = $this->getEntityManager()->createQuery(
            'SELECT p
            FROM App\Entity\Product p
            WHERE p.price > :price
            ORDER BY p.price ASC'
        )->setParameter('price', $price);

        return $query->getResult();
    }
}
```

## QueryBuilder (preferred for dynamic queries)

Use the QueryBuilder when the query is assembled from PHP conditions.

```php
public function findAllGreaterThanPrice(int $price, bool $includeUnavailable = false): array
{
    $qb = $this->createQueryBuilder('p') // 'p' is the alias
        ->where('p.price > :price')
        ->setParameter('price', $price)
        ->orderBy('p.price', 'ASC');

    if (!$includeUnavailable) {
        $qb->andWhere('p.available = TRUE');
    }

    return $qb->getQuery()->execute();

    // single result:
    // return $qb->getQuery()->setMaxResults(1)->getOneOrNullResult();
}
```

## Raw SQL

Returns raw rows (arrays), not entities, unless you use NativeQuery.

```php
public function findAllGreaterThanPrice(int $price): array
{
    $conn = $this->getEntityManager()->getConnection();
    $sql = 'SELECT * FROM product p WHERE p.price > :price ORDER BY p.price ASC';
    $resultSet = $conn->executeQuery($sql, ['price' => $price]);

    return $resultSet->fetchAllAssociative();
}
```

## Join fetch to avoid N+1

When you know you'll need the related object, fetch it in one query instead of
triggering a lazy-load.

```php
public function findOneByIdJoinedToCategory(int $productId): ?Product
{
    $query = $this->getEntityManager()->createQuery(
        'SELECT p, c
        FROM App\Entity\Product p
        INNER JOIN p.category c
        WHERE p.id = :id'
    )->setParameter('id', $productId);

    return $query->getOneOrNullResult();
}
```

Calling `$product->getCategory()` afterwards makes no extra query.

## Usage

```php
// repositories are autowired by type-hint
$products = $productRepository->findAllGreaterThanPrice(1000);
```
