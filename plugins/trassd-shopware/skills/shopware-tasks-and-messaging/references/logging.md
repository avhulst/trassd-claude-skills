# Plugin logging — dedicated Monolog channel

A plugin should log to its own Monolog channel so project owners can redirect it
without touching plugin code.

## 1. Load package configuration in the plugin base class

A Symfony bundle must explicitly load `Resources/config/packages`. Wire a config
loader in the plugin base class `build()` (alternatively use a bundle
extension):

```php
<?php declare(strict_types=1);

namespace Swag\Example;

use Shopware\Core\Framework\Plugin;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\Config\Loader\DelegatingLoader;
use Symfony\Component\Config\Loader\LoaderResolver;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\DirectoryLoader;
use Symfony\Component\DependencyInjection\Loader\GlobFileLoader;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

class SwagExample extends Plugin
{
    public function build(ContainerBuilder $container): void
    {
        parent::build($container);

        $locator = new FileLocator('Resources/config');
        $resolver = new LoaderResolver([
            new YamlFileLoader($container, $locator),
            new GlobFileLoader($container, $locator),
            new DirectoryLoader($container, $locator),
        ]);
        $configLoader = new DelegatingLoader($resolver);

        $confDir = \rtrim($this->getPath(), '/') . '/Resources/config';
        $configLoader->load($confDir . '/{packages}/*.yaml', 'glob');
    }
}
```

## 2. Declare the channel (and optional handler)

In `src/Resources/config/packages/monolog.yaml`, use a unique channel name that
identifies your plugin:

```yaml
monolog:
  channels: ['my_plugin_channel']

  handlers:
    myPluginLogHandler:
        type: rotating_file
        path: "%kernel.logs_dir%/my_plugin_%kernel.environment%.log"
        level: error
        channels: ["my_plugin_channel"]
```

## 3. Inject the logger

Monolog auto-registers a channel-scoped logger service. Inject it by service ID:

```
monolog.logger.my_plugin_channel
```

Using a dedicated channel (rather than the default `logger`) lets project owners
redirect your channel to a handler that suits their setup.
