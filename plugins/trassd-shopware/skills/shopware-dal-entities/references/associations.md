# Associations (classic EntityDefinition)

All examples use two entities `Foo` and `Bar`. The owning ("many") side always
needs an `FkField` for the foreign-key column. The trailing boolean on
association fields is `autoload` and defaults to `false`.

## OneToOne

Foreign key lives on one side (here `foo_id` in `bar`).

```php
// BarDefinition::defineFields()
(new IdField('id', 'id'))->addFlags(new Required(), new PrimaryKey()),
(new FkField('foo_id', 'fooId', FooDefinition::class))->addFlags(new Required()),
new OneToOneAssociationField('foo', 'foo_id', 'id', FooDefinition::class, false),

// FooDefinition::defineFields() — inverse side, no FkField needed
(new OneToOneAssociationField('bar', 'id', 'foo_id', BarDefinition::class, false)),
```

## OneToMany / ManyToOne

A `bar` has many `foo`s; the foreign key (`bar_id`) lives on the "many" side
(`foo`).

```php
// BarDefinition (the "one" side)
new OneToManyAssociationField('foos', FooDefinition::class, 'bar_id'),

// FooDefinition (the "many" side)
(new FkField('bar_id', 'barId', BarDefinition::class))->addFlags(new Required()),
new ManyToOneAssociationField('bar', 'bar_id', BarDefinition::class, 'id'),
```

`OneToManyAssociationField(propertyName, referenceClass, referenceColumn)`.
`ManyToOneAssociationField(propertyName, storageName, referenceClass, referenceField)`.

## ManyToMany (needs a mapping definition)

A third class extending `MappingEntityDefinition` with its own table links both
sides. It holds two `FkField`s (both flagged `PrimaryKey`) and two
`ManyToOneAssociationField`s.

```php
class FooBarMappingDefinition extends MappingEntityDefinition
{
    public const ENTITY_NAME = 'foo_bar';

    public function getEntityName(): string { return self::ENTITY_NAME; }

    protected function defineFields(): FieldCollection
    {
        return new FieldCollection([
            (new FkField('bar_id', 'barId', BarDefinition::class))->addFlags(new PrimaryKey(), new Required()),
            (new FkField('foo_id', 'fooId', FooDefinition::class))->addFlags(new PrimaryKey(), new Required()),
            new ManyToOneAssociationField('bar', 'bar_id', BarDefinition::class, 'id'),
            new ManyToOneAssociationField('foo', 'foo_id', FooDefinition::class, 'id'),
        ]);
    }
}
```

Then add a `ManyToManyAssociationField` to each main definition:

```php
// BarDefinition
new ManyToManyAssociationField(
    'foos',                          // propertyName
    FooDefinition::class,            // referenceDefinition
    FooBarMappingDefinition::class,  // mappingDefinition
    'bar_id',                        // mappingLocalColumn
    'foo_id'                         // mappingReferenceColumn
),

// FooDefinition (mirror columns)
new ManyToManyAssociationField('bars', BarDefinition::class, FooBarMappingDefinition::class, 'foo_id', 'bar_id'),
```

Register the mapping definition with the `shopware.entity.definition` tag like
any other definition. Mapping entities are read with `searchIds()`, not
`search()`.

## autoload caveat

Setting `autoload: true` on **both** an `EntityExtension` and its
`EntityDefinition` leads to recursion / out-of-memory. If you need an
association loaded on every load, set `autoload: true` only on the
`EntityExtension`.
