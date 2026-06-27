# Making your own entity extensible (Persistence bundle)

Example: an extensible `Book` entity in a `SuluBookBundle`. Built on four
classes plus three wiring points so downstream bundles can replace it.

## 1. Entity classes

### BookInterface

`Sulu\Bundle\BookBundle\Entity\BookInterface` (interface). Used as the type of
**every** variable, Doctrine relation and other usage of the entity, so the
implementation stays exchangeable.

### Book

`Sulu\Bundle\BookBundle\Entity\Book` implements `BookInterface`. The default
implementation and the base class for extension; mapped as a **mapped
superclass** (`Book.orm.xml`).

## 2. Repository classes

### BookRepositoryInterface

`Sulu\Bundle\BookBundle\Entity\BookRepositoryInterface` extends the Persistence
`RepositoryInterface`. That base interface declares `createNew()`, which must be
used instead of the constructor so you never create an instance of the wrong
implementation when the entity is swapped.

### BookRepository

`Sulu\Bundle\BookBundle\Entity\BookRepository` implements
`BookRepositoryInterface` and should extend the Persistence `EntityRepository`,
which provides a dynamic `createNew()` returning the currently configured
implementation.

## 3. Configuration — default model/repository

```php
<?php
class Configuration implements ConfigurationInterface
{
    public function getConfigTreeBuilder()
    {
        $treeBuilder = new TreeBuilder();
        $rootNode = $treeBuilder->root('sulu_book')
            // ...
            ->children()
                ->arrayNode('objects')
                    ->addDefaultsIfNotSet()
                    ->children()
                        ->arrayNode('book')
                            ->addDefaultsIfNotSet()
                            ->children()
                                ->scalarNode('model')
                                    ->defaultValue('Sulu\Bundle\BookBundle\Entity\Book')->end()
                                ->scalarNode('repository')
                                    ->defaultValue('Sulu\Bundle\BookBundle\Entity\BookRepository')->end()
                            ->end()
                        ->end()
                    ->end()
                ->end()
            ->end();

        return $treeBuilder;
    }
}
```

This exposes override paths `sulu_book.objects.book.model` and
`sulu_book.objects.book.repository`.

## 4. Extension — register services + autowire alias

```php
<?php
class SuluBookExtension extends Extension
{
    use PersistenceExtensionTrait;

    public function load(array $configs, ContainerBuilder $container)
    {
        $configuration = new Configuration();
        $config = $this->processConfiguration($configuration, $configs);
        // ...
        $this->configurePersistence($config['objects'], $container);
        $container->addAliases([
            'Sulu\Bundle\BookBundle\Entity\BookRepositoryInterface' => 'sulu.repository.book',
        ]);
    }
}
```

## 5. Bundle — resolve interface to implementation in Doctrine

```php
<?php
class SuluBookBundle extends Bundle
{
    use PersistenceBundleTrait;

    public function build(ContainerBuilder $container)
    {
        // ...
        $this->buildPersistence(
            [
                'Sulu\Bundle\BookBundle\Entity\BookInterface' => 'sulu.model.book.class',
            ],
            $container,
        );
    }
}
```

`buildPersistence()` adds a `ResolveTargetEntitiesPass` so the interface
resolves to the configured class in Doctrine mappings.

## What the Persistence bundle provides

For `book`, after wiring:

- `sulu.model.book.class` — parameter, the configured entity class.
- `sulu.repository.book` — service, the configured repository.
- `Sulu\Bundle\BookBundle\Entity\BookRepositoryInterface` — alias to that
  repository service (autowireable).

## Rules

- Type against `BookInterface` / `BookRepositoryInterface` everywhere.
- Create instances via the repository's `createNew()`, never `new Book()`.
- `Book` is a mapped superclass; the concrete entity is the configured `model`.
