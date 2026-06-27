---
name: shopware-dal-entities
description: >-
  Model data with the Shopware 6 Data Abstraction Layer (DAL) — define custom
  entities the modern attribute way (#[Entity] + field attributes) or the classic
  way (EntityDefinition + EntityCollection + Entity registered via the
  shopware.entity.definition tag), pick field types, wire OneToOne/OneToMany/
  ManyToOne/ManyToMany associations, add translations and custom fields, extend
  core entities with EntityExtension, and read data via EntityRepository +
  Criteria (filters, associations, aggregations, paging). Use this skill when
  adding an entity under a Shopware plugin, extending a core entity such as
  product, or querying the DAL.
---

# Shopware DAL: Entities & Reading Data

Shopware 6 has no ORM; it uses a thin **Data Abstraction Layer (DAL)**. Every
registered entity gets an auto-generated `EntityRepository` (named
`<entity_name>.repository` in the DI container) used for all CRUD and search.

There are two ways to define an entity. **Prefer the attribute-based entity**
for new code (Shopware ≥ 6.6.3.0); use the **classic `EntityDefinition`** when
targeting older versions, when extending core entities, or when you need a
full mapping definition.

## Decision guide

- New plugin entity, Shopware ≥ 6.6.3.0 → attribute-based entity (least boilerplate).
- Older Shopware, or extending a **core** entity → classic `EntityDefinition` / `EntityExtension`.
- Many-to-many → always needs a third **mapping** definition/table.
- Reading anything → inject the entity's `*.repository` and build a `Criteria`.

## Rule 1 — Attribute-based entity (modern)

A single class extending `Entity` carries the schema via PHP attributes. No
separate definition, collection, getters or setters are needed.

```php
use Shopware\Core\Framework\DataAbstractionLayer\Entity;
use Shopware\Core\Framework\DataAbstractionLayer\Attribute\Entity as EntityAttribute;
use Shopware\Core\Framework\DataAbstractionLayer\Attribute\Field;
use Shopware\Core\Framework\DataAbstractionLayer\Attribute\FieldType;
use Shopware\Core\Framework\DataAbstractionLayer\Attribute\PrimaryKey;

#[EntityAttribute('example_entity')]
class ExampleEntity extends Entity
{
    #[PrimaryKey]
    #[Field(type: FieldType::UUID)]
    public string $id;

    #[Field(type: FieldType::STRING)]
    public string $name;

    #[Field(type: FieldType::BOOL)]
    public ?bool $active = null;
}
```

Register it in `services.php` with the **`shopware.entity`** tag — Shopware then
auto-creates the `example_entity.definition` and `example_entity.repository`:

```php
$services->set(Examples\ExampleEntity::class)->tag('shopware.entity');
```

Key points:
- The `#[Entity('name')]` `name` is required and must be unique.
- Properties are **public**; no getters/setters. Optional `collectionClass:`
  (≥ 6.6.9.0) sets a custom `EntityCollection`, else the generic one is used.
- A field is **required** unless its type is nullable; force it with `#[Required]`.
- Hide from API by default; expose with `#[Field(..., api: true)]` or a scope
  array (`[AdminApiSource::class]` / `[SalesChannelApiSource::class]`).

More on field types (`FieldType` constants, `PriceField::class`, `#[Serialized]`,
`#[AutoIncrement]`, `#[ForeignKey]`, `#[State]`), JSON, custom fields and the
full attribute association example: see
[references/attribute-entity.md](references/attribute-entity.md).

## Rule 2 — Classic EntityDefinition (definition + entity + collection)

Three classes plus a migration that creates the table (entity name == table
name; `created_at` / `updated_at` are added automatically by the DAL).

1. **Definition** extends `EntityDefinition`, implements `getEntityName()` and
   `defineFields(): FieldCollection`. Optionally override `getEntityClass()`
   and `getCollectionClass()`.
2. **Entity** extends `Entity`, uses `EntityIdTrait`, has `protected` properties
   with getters/setters (properties must be **at least protected**, never
   `readonly`, or the DAL cannot hydrate them).
3. **Collection** extends `EntityCollection`, returns the entity FQCN from
   `getExpectedClass()`.

```php
class ExampleDefinition extends EntityDefinition
{
    public const ENTITY_NAME = 'swag_example';

    public function getEntityName(): string { return self::ENTITY_NAME; }
    public function getEntityClass(): string { return ExampleEntity::class; }
    public function getCollectionClass(): string { return ExampleCollection::class; }

    protected function defineFields(): FieldCollection
    {
        return new FieldCollection([
            (new IdField('id', 'id'))->addFlags(new Required(), new PrimaryKey()),
            new StringField('name', 'name'),
            new BoolField('active', 'active'),
        ]);
    }
}
```

Register with the **`shopware.entity.definition`** tag, passing the technical
entity name; the translation definition must be registered **after** its parent:

```php
$services->set(ExampleDefinition::class)
    ->tag('shopware.entity.definition', ['entity' => 'swag_example']);
```

Field classes take `('storage_name', 'propertyName')` — storage in
snake_case, property in lowerCamelCase. Flags (`Required`, `PrimaryKey`,
`ApiAware`, `CascadeDelete`, …) are added via `->addFlags(...)`.

Full three-class skeleton: see
[references/classic-definition.md](references/classic-definition.md).

## Rule 3 — Associations

Four types: `OneToOne`, `OneToMany`, `ManyToOne`, `ManyToMany`. With the classic
DAL the "many/owning" side needs an `FkField` for the foreign key column.

- **OneToOne / OneToMany–ManyToOne**: owning side has `FkField` + the
  association field (`OneToOneAssociationField` / `ManyToOneAssociationField`);
  inverse side has `OneToManyAssociationField` / `OneToOneAssociationField`.
- **ManyToMany**: requires a separate mapping class extending
  `MappingEntityDefinition` (two `FkField`s flagged `PrimaryKey`, two
  `ManyToOneAssociationField`s) and a `ManyToManyAssociationField` on each side.
- The last `autoload` boolean on association fields defaults to `false`; enable
  sparingly (eager loading can hurt performance). Setting `autoload: true` on
  **both** an `EntityExtension` and its `EntityDefinition` causes recursion /
  out-of-memory — set it on the extension only when needed everywhere.

With attribute entities, associations are nullable array (or `EntityCollection`)
properties carrying `#[OneToMany(entity: '...', ref: '...')]`, `#[ManyToOne]`,
`#[ManyToMany]`, `#[OneToOne]`, plus an `#[ForeignKey(entity: '...')]` id property.

Field signatures and the mapping-definition example: see
[references/associations.md](references/associations.md).

## Rule 4 — Translations

Translated values live in a separate `<entity>_translation` table keyed by
`(<entity>_id, language_id)`.

- **Classic**: create an `EntityTranslationDefinition` (override
  `getParentDefinitionClass()`), an entity extending `TranslationEntity`, and a
  collection. On the parent definition add a `TranslatedField('name')` and a
  `TranslationsAssociationField(ExampleTranslationDefinition::class, '<entity>_id')`.
  Register the translation definition after the parent.
- **Attribute**: mark the field `#[Field(type: ..., translated: true)]`
  (translated fields **must be nullable**; combine with `#[Required]` if needed).
  Shopware auto-creates the `TranslatedField` and `EntityTranslationDefinition`.
  Add a `#[Translations] public ?array $translations` property to load all
  translations via the `translations` association.

Details: see [references/translations.md](references/translations.md).

## Rule 5 — Extending existing (core) entities

To add fields/associations to a core entity (e.g. `product`) **without**
altering its table:

- Create a class extending `EntityExtension`, implement `getDefinitionClass()`
  (e.g. `ProductDefinition::class`) and add fields in `extendFields()`. Register
  with the **`shopware.entity.extension`** tag.
- To **persist** data, store it in your own table and link it via a
  `OneToOneAssociationField` (often `autoload: true` + `CascadeDelete` flag) to a
  separate `EntityDefinition`; create the table via migration with proper FK
  constraints. Write through the parent repo using the association property name.
- To add **runtime-only** data, subscribe to the `*_LOADED_EVENT` (e.g.
  `ProductEvents::PRODUCT_LOADED_EVENT`) and `$entity->addExtension(name, struct)`
  — the value must be a `Struct` (e.g. `ArrayEntity`), not a scalar.
- Many entities at once: `BulkEntityExtension` (≥ 6.6.10.0), tagged
  `shopware.bulk.entity.extension`.
- **Entity extension vs. custom fields**: custom fields are admin-configurable
  and mostly scalar; use an entity extension for associations or technical,
  non-configurable data.

Examples: see [references/extending-entities.md](references/extending-entities.md).

## Rule 6 — Reading data (repository + Criteria)

Inject `<entity>.repository` (typed `EntityRepository`) and search with a
`Criteria`. `search()` returns an `EntitySearchResult` (an iterable
`EntityCollection`); use `->first()` for a single entity.

```php
use Shopware\Core\Framework\Context;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Filter\EqualsFilter;

$criteria = new Criteria();                          // or new Criteria([$id, ...])
$criteria->addFilter(new EqualsFilter('name', 'Example name'));
$criteria->addAssociation('productReviews.customer'); // chainable
$criteria->setLimit(10);
$criteria->setOffset(0);
$criteria->addSorting(new FieldSorting('createdAt', FieldSorting::ASCENDING));

$result = $this->productRepository->search($criteria, $context);
```

- **Filters**: `EqualsFilter`, `RangeFilter` (`RangeFilter::GTE`, …), combine
  with `AndFilter` / `OrFilter` / `NandFilter`. `addPostFilter()` filters the
  result without affecting aggregations.
- **Associations**: `addAssociation('rel')` to load; `getAssociation('rel')`
  returns a nested `Criteria` to filter the association itself.
- **Aggregations**: `addAggregation(new AvgAggregation('avg-rating',
  'productReviews.points'))`, read via `$result->getAggregations()->get(...)`.
- **Mapping (ManyToMany) entities** cannot be loaded with `search()` — use
  `searchIds()` (they are only two primary keys).
- **Large result sets**: use `RepositoryIterator` to fetch in batches; sort by a
  **deterministic** key (add a secondary `FieldSorting('id')` when the primary
  sort isn't unique) to avoid duplicated/skipped rows.

Field and filter reference details: see
[references/reading-data.md](references/reading-data.md).

## Anti-patterns

- Adding a column directly to a core table instead of using `EntityExtension` +
  own table.
- `readonly`/`private` properties on `Entity`/`Struct` classes — the DAL can't hydrate them.
- Non-nullable translated fields (translated fields must be nullable).
- Eager `autoload: true` everywhere — risks recursion and performance issues.
- Using `search()` on a mapping entity (use `searchIds()`).
- Reading huge datasets in one `search()` — use `RepositoryIterator`.
