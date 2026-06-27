# Entity translations (classic)

Translated values live in a `<entity>_translation` table. For `swag_example`
that is `swag_example_translation`.

## Translation table

Primary key `(swag_example_id, language_id)`; both are foreign keys (to the
parent entity and to `language`). Add the translated columns (e.g. `name`) plus
`created_at` / `updated_at`.

```php
CREATE TABLE IF NOT EXISTS `swag_example_translation` (
    `swag_example_id` BINARY(16) NOT NULL,
    `language_id` BINARY(16) NOT NULL,
    `name` VARCHAR(255),
    `created_at` DATETIME(3) NOT NULL,
    `updated_at` DATETIME(3) NULL,
    PRIMARY KEY (`swag_example_id`, `language_id`),
    CONSTRAINT `fk.swag_example_translation.swag_example_id` FOREIGN KEY (`swag_example_id`)
        REFERENCES `swag_example` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
    CONSTRAINT `fk.swag_example_translation.language_id` FOREIGN KEY (`language_id`)
        REFERENCES `language` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
```

## Translation definition

Extends `EntityTranslationDefinition` and overrides `getParentDefinitionClass()`.
`language_id` and the base fields are added automatically.

```php
use Shopware\Core\Framework\DataAbstractionLayer\EntityTranslationDefinition;

class ExampleTranslationDefinition extends EntityTranslationDefinition
{
    public const ENTITY_NAME = 'swag_example_translation';

    public function getEntityName(): string { return self::ENTITY_NAME; }
    public function getParentDefinitionClass(): string { return ExampleDefinition::class; }
    public function getEntityClass(): string { return ExampleTranslationEntity::class; }

    protected function defineFields(): FieldCollection
    {
        return new FieldCollection([
            (new StringField('name', 'name'))->addFlags(new Required()),
        ]);
    }
}
```

## Translation entity & collection

The entity extends `TranslationEntity` (which provides `language_id`
getters/setters). Add the parent id, the translated value, and the parent
association, each with getters/setters.

```php
use Shopware\Core\Framework\DataAbstractionLayer\TranslationEntity;

class ExampleTranslationEntity extends TranslationEntity
{
    protected string $exampleId;
    protected ?string $name;
    protected ExampleEntity $example;
    // getters/setters ...
}
```

Create an `ExampleTranslationCollection` extending `EntityCollection` returning
`ExampleTranslationEntity::class` from `getExpectedClass()`.

## Parent definition

Add a `TranslatedField` for each translated value and a
`TranslationsAssociationField` referencing the translation definition:

```php
(new TranslatedField('name'))->addFlags(new ApiAware(), new Required()),
(new TranslationsAssociationField(
    ExampleTranslationDefinition::class,
    'swag_example_id'
))->addFlags(new ApiAware(), new Required()),
```

## Registration order

Register the translation definition **after** the parent in `services.php`:

```php
$services->set(ExampleDefinition::class)
    ->tag('shopware.entity.definition', ['entity' => 'swag_example']);
$services->set(ExampleTranslationDefinition::class)
    ->tag('shopware.entity.definition', ['entity' => 'swag_example_translation']);
```

## Attribute alternative

With attribute entities, mark the field `#[Field(type: ..., translated: true)]`
(must be nullable) and Shopware auto-creates the `TranslatedField` and
`EntityTranslationDefinition`. Add a nullable `#[Translations] public ?array
$translations` property to load all translations via the `translations`
association. See `attribute-entity.md`.
