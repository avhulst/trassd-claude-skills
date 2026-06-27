# Attribute-based entities (Shopware ≥ 6.6.3.0)

Register the class in `services.php` with the `shopware.entity` tag; Shopware
auto-generates `<name>.definition` and `<name>.repository`.

## Field types

`#[Field(type: ...)]` `type` is a `FieldType` constant. Available constants
(verified in `Attribute/FieldType.php`):

`UUID`, `STRING`, `TEXT`, `INT`, `FLOAT`, `BOOL`, `ENUM`, `JSON`, `DATETIME`,
`DATE`, `DATE_INTERVAL`, `TIME_ZONE`, `EMAIL`, `PRICE`.

```php
#[Field(type: FieldType::STRING)]      public string $string;
#[Field(type: FieldType::TEXT)]        public ?string $text = null;
#[Field(type: FieldType::INT)]         public ?int $int;
#[Field(type: FieldType::FLOAT)]       public ?float $float;
#[Field(type: FieldType::BOOL)]        public ?bool $bool;
#[Field(type: FieldType::DATETIME)]    public ?\DateTimeImmutable $datetime = null;
#[Field(type: FieldType::JSON)]        public ?array $json = null;
```

### Field class directly (≥ 6.6.9.0)

Pass any `Field` subclass FQCN as the `type`:

```php
use Shopware\Core\Framework\DataAbstractionLayer\Field\PriceField;
use Shopware\Core\Framework\DataAbstractionLayer\Pricing\PriceCollection;

#[Field(type: PriceField::class)]
public ?PriceCollection $price = null;
```

### Special field attributes

```php
#[AutoIncrement]
public int $autoIncrement;

#[ForeignKey(entity: 'currency')]
public ?string $foreignKey;

#[State(machine: OrderStates::STATE_MACHINE)]
public ?string $stateId = null;
```

### JSON with custom serializer

```php
use Shopware\Core\Framework\DataAbstractionLayer\Attribute\Serialized;
use Shopware\Core\Framework\DataAbstractionLayer\FieldSerializer\PriceFieldSerializer;

#[Serialized(serializer: PriceFieldSerializer::class)]
public ?PriceCollection $serialized = null;
```

### Column name override

```php
#[Field(type: FieldType::STRING, column: 'another_column_name')]
public ?string $differentName = null;
```

## Custom fields

Either use the trait for ready-made helpers:

```php
use Shopware\Core\Framework\DataAbstractionLayer\EntityCustomFieldsTrait;

class ExampleEntity extends Entity
{
    use EntityCustomFieldsTrait;
    // ...
}
```

Or take full control with the attribute:

```php
use Shopware\Core\Framework\DataAbstractionLayer\Attribute\CustomFields;

#[CustomFields]
public ?array $customFields = null;
```

## API encoding

Fields are not exposed in the API by default.

```php
#[Field(type: FieldType::STRING, api: true)]                          public string $everywhere;
#[Field(type: FieldType::STRING, api: [AdminApiSource::class])]       public string $adminOnly;
#[Field(type: FieldType::STRING, api: [SalesChannelApiSource::class])] public string $storeOnly;
```

## Required fields

A field is required unless nullable; force with `#[Required]`. Translated
fields must be nullable, so mark them required explicitly:

```php
#[Required]
#[Field(type: FieldType::STRING, translated: true)]
public ?string $required = null;
```

## Associations (attribute form)

```php
use Shopware\Core\Framework\DataAbstractionLayer\Attribute\{ForeignKey, ManyToOne, OneToOne, OneToMany, ManyToMany, OnDelete};
use Shopware\Core\System\Currency\CurrencyEntity;

#[ForeignKey(entity: 'currency')]
public ?string $currencyId = null;

#[ManyToOne(entity: 'currency', onDelete: OnDelete::RESTRICT)]
public ?CurrencyEntity $currency = null;

#[OneToOne(entity: 'currency', onDelete: OnDelete::SET_NULL)]
public ?CurrencyEntity $follow = null;

/** @var array<string, ExampleEntityAgg>|null */
#[OneToMany(entity: 'example_entity_agg', ref: 'example_entity_id', onDelete: OnDelete::CASCADE)]
public ?array $aggs = null;

/** @var array<string, CurrencyEntity>|null */
#[ManyToMany(entity: 'currency', onDelete: OnDelete::CASCADE)]
public ?array $currencies = null;
```

Associations are nullable array properties (key = associated entity ID, value =
the entity) or may be typed as `EntityCollection`. `OnDelete` options include
`RESTRICT`, `SET_NULL`, `CASCADE`.

## Translations

`translated: true` auto-creates a `TranslatedField` + `EntityTranslationDefinition`.
Add a `#[Translations]` property (nullable) to load all translations through the
`translations` association on the criteria.

```php
use Shopware\Core\Framework\DataAbstractionLayer\Attribute\Translations;

#[Field(type: FieldType::STRING, translated: true)]
public ?string $string = null;

/** @var array<string, ArrayEntity>|null */
#[Translations]
public ?array $translations = null;
```

## Optional custom collection

```php
#[EntityAttribute('example_entity', collectionClass: ExampleEntityCollection::class)]

/**
 * @extends EntityCollection<ExampleEntity>
 */
class ExampleEntityCollection extends EntityCollection
{
    protected function getExpectedClass(): string
    {
        return ExampleEntity::class;
    }
}
```
