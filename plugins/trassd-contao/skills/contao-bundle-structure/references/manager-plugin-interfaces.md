# Manager Plugin interfaces — details

The Manager Plugin is a single class that may implement several interfaces from
the `Contao\ManagerPlugin\…` namespace. Implement only what you need.

## BundlePluginInterface — load order & replacement

`getBundles(ParserInterface $parser): array` returns `BundleConfig` objects.

- `BundleConfig::create(MyBundle::class)` registers the bundle.
- `->setLoadAfter([ContaoCoreBundle::class, 'legacy_module'])` — load after
  other bundle classes and/or after a legacy `system/modules/<name>` module
  (the internal name is the module's folder name).
- `->setReplace(['old_module_name'])` — declare that this bundle replaces a
  legacy module, so other legacy modules that depend on it still load after it.

```php
return [
    BundleConfig::create(SomeBundle::class)
        ->setLoadAfter([ContaoCoreBundle::class, 'notification_center'])
        ->setReplace(['old_module_name']),
];
```

## RoutingPluginInterface — register routes

A bundle's routes are not auto-loaded; expose them here.

```php
use Contao\ManagerPlugin\Routing\RoutingPluginInterface;
use Symfony\Component\Config\Loader\LoaderResolverInterface;
use Symfony\Component\HttpKernel\KernelInterface;

class Plugin implements RoutingPluginInterface
{
    public function getRouteCollection(LoaderResolverInterface $resolver, KernelInterface $kernel)
    {
        return $resolver
            ->resolve(__DIR__.'/../../config/routes.yaml')
            ->load(__DIR__.'/../../config/routes.yaml');
    }
}
```

With `config/routes.yaml`:

```yaml
somevendor.contao_example_bundle.controller:
    resource: ../src/Controller
    type: attribute
```

If you only use attribute routing and keep all controllers in `src/Controller`,
skip the YAML file and resolve the directory directly:

```php
return $resolver
    ->resolve(__DIR__.'/../Controller', 'attribute')
    ->load(__DIR__.'/../Controller');
```

## ConfigPluginInterface — load bundle config

```php
use Contao\ManagerPlugin\Config\ConfigPluginInterface;
use Symfony\Component\Config\Loader\LoaderInterface;

class Plugin implements ConfigPluginInterface
{
    public function registerContainerConfiguration(LoaderInterface $loader, array $config)
    {
        $loader->load(__DIR__.'/../../config/config.yaml');
    }
}
```

## ExtensionPluginInterface — override other bundles' config

Only needed for the edge case where config **merge order** matters and a plain
`config.yaml` cannot express it — e.g. inserting a firewall before Contao's
`frontend` firewall, or adding a Monolog handler before the `contao` handler.
Nodes like `security.firewalls` forbid being defined twice, so you must mutate
the already-collected extension configs instead.

```php
use Contao\ManagerPlugin\Config\ContainerBuilder;
use Contao\ManagerPlugin\Config\ExtensionPluginInterface;

class Plugin implements ExtensionPluginInterface
{
    public function getExtensionConfig($extensionName, array $extensionConfigs, ContainerBuilder $container)
    {
        if ('security' !== $extensionName) {
            return $extensionConfigs;
        }
        // splice your firewall in front of the 'frontend' firewall here,
        // then return the modified $extensionConfigs.
        return $extensionConfigs;
    }
}
```

Notes:
- You may receive several config arrays and have to apply the change to each.
- You **cannot** add a compiler pass here (`getExtensionConfig()` runs during
  Symfony's `MergeExtensionConfigurationPass`). Use a normal compiler pass for that.

## DependentPluginInterface — order plugins (not bundles)

Ensure another package's Manager Plugin loads before yours:

```php
use Contao\ManagerPlugin\Dependency\DependentPluginInterface;

class Plugin implements DependentPluginInterface
{
    public function getPackageDependencies()
    {
        return ['contao/news-bundle'];
    }
}
```

## HttpCacheSubscriberPluginInterface — modify HttpCache

Return event subscribers that adjust requests/responses before the Symfony
HttpCache (e.g. strip cookies, add headers).

```php
use Contao\ManagerPlugin\Routing\HttpCacheSubscriberPluginInterface;

class Plugin implements HttpCacheSubscriberPluginInterface
{
    public function getHttpCacheSubscribers(): array
    {
        return [new CustomCacheSubscriber()];
    }
}
```

## Alternative service loading: a DependencyInjection Extension class

Instead of importing `services.yaml` from `loadExtension()`, you can use a
classic DI extension. The class name is the bundle name with `Bundle` replaced
by `Extension`, placed in the `DependencyInjection` sub-namespace.

```php
// src/DependencyInjection/ContaoExampleExtension.php
namespace Somevendor\ContaoExampleBundle\DependencyInjection;

use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\Extension;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

class ContaoExampleExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container): void
    {
        (new YamlFileLoader($container, new FileLocator(__DIR__.'/../../config')))
            ->load('services.yaml');
    }
}
```

When the bundle extends `AbstractBundle`, this is not picked up automatically —
override `getContainerExtension()` in the bundle class to return a new instance
of the extension.
