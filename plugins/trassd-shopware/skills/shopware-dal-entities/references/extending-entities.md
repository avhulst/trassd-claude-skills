# Extending existing (core) entities

Add fields/associations to a core entity (e.g. `product`) without altering its
table. Place the extension under `src/Extension/<Domain>/<Entity>/`.

## Extension class

Extends `EntityExtension`, points at the target definition, adds fields in
`extendFields()`:

```php
use Shopware\Core\Content\Product\ProductDefinition;
use Shopware\Core\Framework\DataAbstractionLayer\EntityExtension;
use Shopware\Core\Framework\DataAbstractionLayer\FieldCollection;

class CustomExtension extends EntityExtension
{
    public function extendFields(FieldCollection $collection): void
    {
        $collection->add(
            // new fields here
        );
    }

    public function getDefinitionClass(): string
    {
        return ProductDefinition::class;
    }
}
```

Register with the `shopware.entity.extension` tag:

```php
$services->set(CustomExtension::class)->tag('shopware.entity.extension');
```

## Persisting data (own table + OneToOne)

You must not add a column to the core table. Store data in your own table and
link it with a `OneToOneAssociationField` (usually `autoload: true` plus the
`CascadeDelete` flag):

```php
use Shopware\Core\Framework\DataAbstractionLayer\Field\Flag\CascadeDelete;
use Shopware\Core\Framework\DataAbstractionLayer\Field\OneToOneAssociationField;

$collection->add(
    (new OneToOneAssociationField('exampleExtension', 'id', 'product_id', ExampleExtensionDefinition::class, true))
        ->addFlags(new CascadeDelete())
);
```

Create the backing `ExampleExtensionDefinition` (a normal `EntityDefinition`)
with an `FkField('product_id', 'productId', ProductDefinition::class)`, the new
field(s), and the inverse `OneToOneAssociationField`. For **versioned** core
entities (like product) also add `ReferenceVersionField(ProductDefinition::class,
'product_version_id')` and a matching `product_version_id` column. Register the
definition with the `shopware.entity.definition` tag and create the table via a
migration with proper FK constraints.

Write through the parent repository using the association property name:

```php
$this->productRepository->upsert([[
    'id' => '<product id>',
    'exampleExtension' => ['customString' => 'foo bar'],
]], $context);
```

## Runtime-only data (no database)

Subscribe to the entity's loaded event and attach a `Struct` (e.g.
`ArrayEntity`) via `addExtension()` — the value must be a struct, not a scalar:

```php
use Shopware\Core\Content\Product\ProductEvents;
use Shopware\Core\Framework\Struct\ArrayEntity;

public static function getSubscribedEvents(): array
{
    return [ProductEvents::PRODUCT_LOADED_EVENT => 'onProductsLoaded'];
}

public function onProductsLoaded(EntityLoadedEvent $event): void
{
    foreach ($event->getEntities() as $product) {
        $product->addExtension('custom_string', new ArrayEntity(['foo' => 'bar']));
    }
}
```

Register the subscriber with the `kernel.event_subscriber` tag.

## Bulk extensions (≥ 6.6.10.0)

Extend many entities in one class via `BulkEntityExtension::collect()`, yielding
`entityName => [fields...]`. Register with the `shopware.bulk.entity.extension`
tag.

```php
public function collect(): \Generator
{
    yield ProductDefinition::ENTITY_NAME => [
        new FkField('main_category_id', 'mainCategoryId', CategoryDefinition::class),
    ];
    yield CategoryDefinition::ENTITY_NAME => [
        new FkField('product_id', 'productId', ProductDefinition::class),
        new ManyToOneAssociationField('product', 'product_id', ProductDefinition::class),
    ];
}
```

## Extension vs. custom fields

- **Custom fields**: admin-configurable, mostly scalar values.
- **Entity extension**: for associations or technical, non-configurable data
  (can also hold scalar values without an association).
