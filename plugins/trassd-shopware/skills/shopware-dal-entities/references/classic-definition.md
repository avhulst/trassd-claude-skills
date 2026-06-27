# Classic EntityDefinition (definition + entity + collection)

Use when targeting older Shopware versions or extending core entities. The
entity name equals the table name; `created_at` / `updated_at` are added
automatically and must not be declared. The repository is registered as
`<entity_name>.repository`.

## 1. Migration (creates the table)

Always prefix plugin tables (e.g. manufacturer name). Create the table in a
`MigrationStep`:

```php
CREATE TABLE IF NOT EXISTS `swag_example` (
    `id` BINARY(16) NOT NULL,
    `name` VARCHAR(255) COLLATE utf8mb4_unicode_ci,
    `description` VARCHAR(255) COLLATE utf8mb4_unicode_ci,
    `active` TINYINT(1) COLLATE utf8mb4_unicode_ci,
    `created_at` DATETIME(3) NOT NULL,
    `updated_at` DATETIME(3),
    PRIMARY KEY (`id`)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
```

## 2. EntityDefinition

```php
use Shopware\Core\Framework\DataAbstractionLayer\EntityDefinition;
use Shopware\Core\Framework\DataAbstractionLayer\FieldCollection;
use Shopware\Core\Framework\DataAbstractionLayer\Field\BoolField;
use Shopware\Core\Framework\DataAbstractionLayer\Field\IdField;
use Shopware\Core\Framework\DataAbstractionLayer\Field\StringField;
use Shopware\Core\Framework\DataAbstractionLayer\Field\Flag\PrimaryKey;
use Shopware\Core\Framework\DataAbstractionLayer\Field\Flag\Required;

class ExampleDefinition extends EntityDefinition
{
    public const ENTITY_NAME = 'swag_example';

    public function getEntityName(): string { return self::ENTITY_NAME; }

    // Optional: return your own classes instead of the generic ArrayEntity / EntityCollection
    public function getEntityClass(): string { return ExampleEntity::class; }
    public function getCollectionClass(): string { return ExampleCollection::class; }

    protected function defineFields(): FieldCollection
    {
        return new FieldCollection([
            (new IdField('id', 'id'))->addFlags(new Required(), new PrimaryKey()),
            new StringField('name', 'name'),
            new StringField('description', 'description'),
            new BoolField('active', 'active'),
        ]);
    }
}
```

Field constructors take `('storage_name', 'propertyName')`: storage in
snake_case, property in lowerCamelCase. Flags are attached via `->addFlags(...)`
(e.g. `Required`, `PrimaryKey`, `ApiAware`, `CascadeDelete`).

## 3. Entity class

Properties must be **at least `protected`** (never `readonly`) so the DAL can
hydrate them. `EntityIdTrait` supplies the `id`.

```php
use Shopware\Core\Framework\DataAbstractionLayer\Entity;
use Shopware\Core\Framework\DataAbstractionLayer\EntityIdTrait;

class ExampleEntity extends Entity
{
    use EntityIdTrait;

    protected ?string $name;
    protected ?string $description;
    protected bool $active;

    public function getName(): ?string { return $this->name; }
    public function setName(?string $name): void { $this->name = $name; }
    // ... getters/setters for the remaining fields
}
```

## 4. EntityCollection

```php
use Shopware\Core\Framework\DataAbstractionLayer\EntityCollection;

/**
 * @extends EntityCollection<ExampleEntity>
 */
class ExampleCollection extends EntityCollection
{
    protected function getExpectedClass(): string
    {
        return ExampleEntity::class;
    }
}
```

## 5. Register (services.php)

Use the `shopware.entity.definition` tag with the technical entity name:

```php
$services->set(ExampleDefinition::class)
    ->tag('shopware.entity.definition', ['entity' => 'swag_example']);
```

If you omit the custom `Entity`/`EntityCollection` classes, Shopware uses the
generic `ArrayEntity` and `EntityCollection` instead.
