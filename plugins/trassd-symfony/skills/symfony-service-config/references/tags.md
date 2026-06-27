# Service tags

A tag marks a service so Symfony or a bundle registers it specially (e.g.
`twig.extension` extensions are collected and handed to Twig). Most of the time
you should let autoconfiguration apply tags for you.

## Autoconfiguring tags (preferred)

With `autoconfigure: true`:

- Implementing a recognized interface auto-applies the matching tag. Example:
  a class implementing `Twig\Extension\ExtensionInterface` gets `twig.extension`
  with no config.
- Attributes such as `#[AsMessageHandler]`, `#[AsEventListener]` and
  `#[AsCommand]` are registered for autoconfiguration and apply their tags.

### Auto-tag your own base types

Apply a tag to every class implementing your interface (or extending a base
class) without repeating it per service.

With the attribute on the interface:

```php
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;

#[AutoconfigureTag('app.custom_tag')]
interface CustomInterface {}
```

`#[AutoconfigureTag]` with no name uses the target's FQCN as the tag name. For
more control (laziness, bindings, calls, public), use `#[Autoconfigure]`.

Or via `_instanceof` in YAML:

```yaml
# config/services.yaml
services:
    _instanceof:
        App\Security\CustomInterface:
            tags: ['app.custom_tag']
```

## Manual tags

Use a manual tag only when no autoconfiguration rule fits, or when you need
per-service tag attributes.

```yaml
# config/services.yaml
services:
    App\Twig\AppExtension:
        tags: ['twig.extension']
```

With additional attributes (e.g. priority, alias, an index key):

```yaml
services:
    App\Handler\One:
        tags:
            - { name: 'app.handler', priority: 20, key: 'handler_one' }
```

In YAML a bare string tag (`tags: ['app.handler']`) is equivalent to
`tags: [{ name: 'app.handler' }]`; the verbose form is only needed for extra
attributes.

## Consuming tagged services

### Inject the whole collection (no compiler pass needed)

Symfony injects every service carrying a tag as an `iterable`. Prefer the
attribute:

```php
use Symfony\Component\DependencyInjection\Attribute\AutowireIterator;

class HandlerCollection
{
    public function __construct(
        #[AutowireIterator('app.handler')]
        iterable $handlers,
    ) {}
}
```

Or in YAML with `!tagged_iterator`:

```yaml
services:
    App\Handler\One:  { tags: ['app.handler'] }
    App\Handler\Two:  { tags: ['app.handler'] }
    App\HandlerCollection:
        arguments:
            - !tagged_iterator app.handler
```

Options:

- **Priority** — higher runs/sorts earlier. Set `priority` on the tag, implement
  static `getDefaultPriority()`, or use `#[AsTaggedItem(priority: ...)]`.
- **Index** — control the array keys with `index_by` / `default_index_method`
  on the iterator, or `#[AsTaggedItem(index: 'handler_one')]` on the class.
  Falls back to the service id.
- **exclude / exclude_self** — drop specific services; the referencing service
  is excluded from its own tag iterable by default.

### Compiler-pass fallback

Only when the shortcut isn't enough (e.g. you must call a method per tagged
service), ask the container for tagged ids in a compiler pass:

```php
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Reference;

class MailTransportPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->has(TransportChain::class)) {
            return;
        }
        $definition = $container->findDefinition(TransportChain::class);

        foreach ($container->findTaggedServiceIds('app.mail_transport') as $id => $tags) {
            $definition->addMethodCall('addTransport', [new Reference($id)]);
        }
    }
}
```

Register it from the Kernel:

```php
protected function build(ContainerBuilder $container): void
{
    $container->addCompilerPass(new MailTransportPass());
}
```

A service can carry the same tag more than once, so iterate the inner `$tags`
array when reading per-tag attributes.

## Tagging non-service classes

Entities/DTOs/value objects should be discoverable at compile time but never
instantiated by the container. Tag them as *resource* tags
(`#[AutoconfigureResourceTag]`, or `addResourceTag()` in a callback registered
via `registerAttributeForAutoconfiguration`) and read them with
`findTaggedResourceIds()` in a compiler pass.
