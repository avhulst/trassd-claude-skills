---
name: doctrine-entity-design
description: >-
  Design Doctrine entities and their associations the Symfony way — mapping with
  attributes, relationships, repositories, and value objects. Triggers when
  creating or changing entities under src/Entity or their repositories.
---

# Doctrine Entity Design

Guidance for modeling Doctrine ORM entities, repositories, and associations in a
Symfony application. Entities are plain PHP objects; Doctrine learns about them
only through mapping metadata.

## Defining entities

- **Use PHP attributes for mapping**, never YAML/XML. Attributes are the
  recommended, most convenient way to declare mapping metadata.
- Mark the class with `#[ORM\Entity(repositoryClass: ...)]` and import
  `use Doctrine\ORM\Mapping as ORM;`.
- Give each entity an identifier with `#[ORM\Id]` + `#[ORM\GeneratedValue]` +
  `#[ORM\Column]`. Use `--with-uuid` / `--with-ulid` (make:entity) to get a
  UUID/ULID id instead of an auto-increment int.
- Map each persisted property with `#[ORM\Column]`. Common options: `length`
  (strings), `nullable`, `type` (e.g. `Types::TEXT`), `unique`, `enumType`.

```php
#[ORM\Entity(repositoryClass: ProductRepository::class)]
class Product
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private ?string $name = null;

    #[ORM\Column]
    private ?int $price = null; // store money as integers (e.g. 1999 = $19.99)

    public function getId(): ?int
    {
        return $this->id;
    }
    // ... getters/setters
}
```

### Field types and value objects

- Pick from Doctrine's mapping types (numbers, strings, dates, JSON, binary…).
- **Backed enums** model a closed set of values: declare the enum, then map with
  `#[ORM\Column(enumType: Suit::class)]`. Only *backed* enums work (Doctrine
  persists their scalar value).
- Symfony bridge value-object types: `UuidType` / `UlidType` for
  `Symfony\Component\Uid\Uuid` / `Ulid`, and `date_point` / `day_point` /
  `time_point` for `Symfony\Component\Clock\DatePoint`. Prefer `date_point` over
  `datetime_immutable` when you want Clock-component testability.
- Avoid reserved SQL keywords for table/column names; override with
  `#[ORM\Table(name: ...)]` or the column `name:` option. A `unique=true` string
  column must cap `length` at 190 to stay under MySQL/InnoDB's index limit.

### Constants for fixed values

Define options that rarely change (e.g. a listing page size) as PHP constants on
the related entity, not as container parameters — constants are usable
everywhere, including Twig and entities:

```php
class Post
{
    public const NUMBER_OF_ITEMS = 10;
}
```

## make:entity workflow

- Scaffold/extend entities with `php bin/console make:entity` (requires
  `symfony/orm-pack` + `symfony/maker-bundle`). Re-run it on an existing entity
  to add fields; it also generates getters/setters.
- Treat generated code as *yours* — edit freely. `--regenerate` regenerates
  missing accessors; add `--overwrite` to regenerate all.
- After any mapping change: `make:migration` then
  `doctrine:migrations:migrate`. Commit migration files and run migrations on
  deploy. (For associations use `doctrine:migrations:diff` to produce the diff.)

## Validation from mapping

The validator can reuse Doctrine metadata via `auto_mapping`: `nullable=false`
→ `NotNull`, `unique=true` → `UniqueEntity`, `length` → `Length`, `type` →
`Type`. This complements, but does not replace, explicit validation constraints.

## Repositories

- Each entity gets a repository extending `ServiceEntityRepository`, wired to the
  entity class via the constructor; reference it from `repositoryClass`.
- Inject repositories by type-hint (autowiring); they are services. Built-in
  finders: `find()`, `findOneBy()`, `findBy()`, `findAll()`.
- Put **complex/reusable queries as named methods on the repository** rather than
  scattering DQL/SQL in controllers. Use the **QueryBuilder** for queries built
  dynamically from PHP conditions; use DQL for static queries; drop to raw SQL
  only when necessary (returns raw rows, not entities).

```php
class ProductRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Product::class);
    }

    /** @return Product[] */
    public function findAllGreaterThanPrice(int $price): array
    {
        return $this->createQueryBuilder('p')
            ->andWhere('p.price > :price')
            ->setParameter('price', $price)
            ->orderBy('p.price', 'ASC')
            ->getQuery()
            ->getResult();
    }
}
```

See [references/repositories.md](references/repositories.md) for DQL, raw SQL,
and join-fetch examples.

## Associations

Choose the relationship first: if **both** sides hold many of the other, use
`ManyToMany`; otherwise `ManyToOne` / `OneToMany` (one association seen from two
sides). `OneToOne` behaves like `ManyToOne` in practice.

### Owning vs inverse side

- The **owning side** is where the foreign key lives — always the `ManyToOne`
  side (for `ManyToMany` you pick the owning side). You **must set the
  relationship on the owning side** for it to persist.
- The owning `#[ORM\ManyToOne]` mapping is **required**. The inverse
  `#[ORM\OneToMany]` is **optional** — add it only if you need to navigate from
  the one side (e.g. `$category->getProducts()`). Linking the two requires
  `inversedBy` (owning) ↔ `mappedBy` (inverse).

```php
// owning side — Product
#[ORM\ManyToOne(targetEntity: Category::class, inversedBy: 'products')]
private ?Category $category = null;

// inverse side — Category
#[ORM\OneToMany(targetEntity: Product::class, mappedBy: 'category')]
private Collection $products;     // init in __construct() as new ArrayCollection()
```

- Inverse collections must be `Doctrine\Common\Collections\Collection`,
  initialized to an `ArrayCollection` in the constructor.
- make:entity generates `addX()` / `removeX()` helpers that keep both sides in
  sync by calling the owning-side setter (e.g. `$product->setCategory($this)`).
  This is what lets you update the relation from the inverse side.
- Set whole *objects*, not ids: `$product->setCategory($category)`. Persisting
  both and `flush()` writes the foreign key.

### Cascade, orphan removal, lazy loading

- `orphanRemoval: true` on the `OneToMany` deletes a child once it is removed
  from the parent collection (vs. just nulling the FK).
- Related objects are **lazily loaded** via proxy objects — accessing
  `$product->getCategory()` triggers a second query. Avoid the N+1 by
  join-fetching (`SELECT p, c ... JOIN p.category c`) when you know you need both.
- Iterating a large inverse collection (e.g. `contains()` checks) can load huge
  result sets — only map the inverse side when you actually need it.

See [references/associations.md](references/associations.md) for full
owning/inverse classes, collection sync helpers, and join-fetch queries.
