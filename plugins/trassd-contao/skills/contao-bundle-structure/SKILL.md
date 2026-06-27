---
name: contao-bundle-structure
description: >-
  Scaffold a Contao extension/bundle the recommended way — the bundle class,
  composer.json (type contao-bundle), the Contao Manager plugin, namespaces,
  service config, and publishing. Use when creating a Contao bundle/extension,
  wiring its ContaoManager Plugin, fixing bundle load order, choosing the
  composer package type, setting up PSR-4 namespaces/services.yaml, or
  publishing a bundle to Packagist/the Contao Manager. Triggers on phrases like
  "new Contao bundle", "contao-bundle composer type", "manager-plugin",
  "BundlePluginInterface", "setLoadAfter", or "publish my Contao extension".
---

# Contao Bundle Structure

Contao is a Symfony-based CMS; a Contao extension **is** a Symfony bundle, with
a few Contao-specific additions. Build it like a Symfony bundle, then add the
two things that make it Contao-aware: the `contao-bundle` composer type and a
**Contao Manager Plugin**.

## When to use

Creating a reusable extension/bundle, wiring its Manager Plugin, fixing bundle
load order, registering services, or publishing. For project-local (non-reusable)
code you usually don't need a bundle — put classes under `src/` in the `App\`
namespace of the Managed Edition (autoloaded automatically) and skip to
[the application-specific plugin](#application-specific-manager-plugin).

## Directory layout

Follow the [Symfony bundle directory structure](https://symfony.com/doc/current/bundles.html#bundle-directory-structure).
PHP under `src/`; Contao-specific resources under `contao/`:

```
my-bundle/
├── composer.json
├── src/
│   ├── ContaoExampleBundle.php          # the bundle class
│   └── ContaoManager/
│       └── Plugin.php                   # the Manager Plugin
├── config/
│   ├── services.yaml
│   └── routes.yaml                      # only if the bundle adds routes
└── contao/                             # Contao resources (all optional)
    ├── dca/                            # Data Container Array definitions
    ├── languages/                      # XLIFF / PHP translations
    └── templates/
```

Contao-specific resource folders are `config`, `dca`, `languages`, `templates`.
Most bundles need little under `contao/` — much of the old `config/config.php`
now lives in the Symfony container, and translations can use the Symfony Translator.

## composer.json

The single defining difference from a Symfony bundle is the `type`:

```json
{
    "name": "somevendor/contao-example-bundle",
    "type": "contao-bundle",
    "require": {
        "contao/core-bundle": "^4.13 || ^5.0"
    },
    "license": "LGPL-3.0-or-later",
    "autoload": {
        "psr-4": {
            "Somevendor\\ContaoExampleBundle\\": "src/"
        }
    },
    "extra": {
        "contao-manager-plugin": "Somevendor\\ContaoExampleBundle\\ContaoManager\\Plugin"
    }
}
```

Rules:

- **`type: contao-bundle`** lets the Contao Manager and `extensions.contao.org`
  discover the package. It is the only hard difference from `symfony-bundle`.
- **`require contao/core-bundle`** at the version range you target (the Contao
  version, e.g. `^4.13 || ^5.0`).
- **Package name** convention: `<vendor>/contao-<name>-bundle` in kebab-case,
  vendor = your Git org name.
- **PSR-4 autoload** maps your top-level namespace (e.g.
  `Somevendor\ContaoExampleBundle\`) to `src/`.
- **`extra.contao-manager-plugin`** is the FQCN of your Manager Plugin (see below).
- Do **not** put `contao/manager-plugin` in `require` — see
  [the conflict/require-dev pattern](#why-not-require-the-manager-plugin).

Generate the skeleton with `composer init` and set the type to `contao-bundle`.
For local development before publishing, add a `path` repository in the root
project's `composer.json` and require the bundle as `dev-main`; see
[references/local-development.md](references/local-development.md).

## The bundle class

Keep it minimal. Extend `AbstractBundle` so the recommended structure applies by
default. The class name is free; conventionally it mirrors the namespace.

```php
// src/ContaoExampleBundle.php
namespace Somevendor\ContaoExampleBundle;

use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class ContaoExampleBundle extends AbstractBundle
{
}
```

## The Contao Manager Plugin

A plain Symfony app loads bundles via its kernel. The **Contao Managed Edition**
doesn't know your bundle exists or what order to load it in — the Manager Plugin
tells it. It is **optional**: without it the bundle still works, but a user must
register it manually and it won't appear in the Contao Manager or on
`extensions.contao.org`. Provide one if you target the Managed Edition.

Best practice: name the class `Plugin` in the `ContaoManager` sub-namespace
(neither is technically required, only the `extra.contao-manager-plugin` FQCN
must resolve). Implement only the plugin interfaces you need.

Minimal plugin that registers the bundle **after** the Contao core bundle (order
matters for DCA, translations, config overrides):

```php
// src/ContaoManager/Plugin.php
namespace Somevendor\ContaoExampleBundle\ContaoManager;

use Contao\CoreBundle\ContaoCoreBundle;
use Contao\ManagerPlugin\Bundle\BundlePluginInterface;
use Contao\ManagerPlugin\Bundle\Config\BundleConfig;
use Contao\ManagerPlugin\Bundle\Parser\ParserInterface;
use Somevendor\ContaoExampleBundle\ContaoExampleBundle;

class Plugin implements BundlePluginInterface
{
    public function getBundles(ParserInterface $parser): array
    {
        return [
            BundleConfig::create(ContaoExampleBundle::class)
                ->setLoadAfter([ContaoCoreBundle::class]),
        ];
    }
}
```

This mirrors what Contao's own bundles do — the news bundle's plugin is the same
shape (`BundleConfig::create(ContaoNewsBundle::class)->setLoadAfter([ContaoCoreBundle::class])`).

### Plugin interfaces

A single `Plugin` class may implement several of these:

| Interface | Purpose |
| --- | --- |
| `Contao\ManagerPlugin\Bundle\BundlePluginInterface` | Register bundle(s) with the kernel and set load order (`setLoadAfter()`, `setReplace()`). |
| `Contao\ManagerPlugin\Config\ConfigPluginInterface` | Load YAML config for your own or another bundle. |
| `Contao\ManagerPlugin\Config\ExtensionPluginInterface` | Modify *other* bundles' extension config where merge order matters (firewalls, monolog handlers). Only for that edge case. |
| `Contao\ManagerPlugin\Dependency\DependentPluginInterface` | Ensure other packages' **plugins** load first. |
| `Contao\ManagerPlugin\Routing\RoutingPluginInterface` | Add routes to the application router. |
| `Contao\ManagerPlugin\Routing\HttpCacheSubscriberPluginInterface` | Add HttpCache event subscribers. |

For routing details and the `setLoadAfter`/`setReplace`/legacy-module nuances,
see [references/manager-plugin-interfaces.md](references/manager-plugin-interfaces.md).

### Why not require the manager-plugin

`contao/manager-plugin` must stay optional so the bundle is still usable in a
plain Symfony app. Declare it as a conflict + dev dependency, never a hard require:

```json
{
    "conflict": { "contao/manager-plugin": "<2.0 || >=3.0" },
    "require-dev": { "contao/manager-plugin": "^2.0" }
}
```

### Application-specific Manager Plugin

For a Managed Edition project (not a reusable bundle), put the plugin at
`\App\ContaoManager\Plugin` — it is auto-loaded, no `extra` key needed. Run
`composer install` afterwards to register it.

## Namespaces

Recommended (none mandatory). In an application use the `App\` root; in a bundle
use your bundle root in the same shape:

| Namespace | Resource |
| --- | --- |
| `…\ContaoManager` | Manager Plugin |
| `…\Controller\ContentElement` | Content element controllers |
| `…\Controller\FrontendModule` | Front end module controllers |
| `…\Cron` | Cron jobs |
| `…\EventListener` | Symfony listeners, Contao hooks & callbacks |
| `…\Model` | Database models |
| `…\Widget` | Form widgets |

Class-name suffixes follow Symfony custom: `*Controller`, `*Cron`, `*Listener`,
`*Model`.

## Service configuration

A bundle must load its own service config (the Managed Edition does **not**
auto-load a bundle's `services.yaml`). With `AbstractBundle`, import it from
`loadExtension()`:

```php
// src/ContaoExampleBundle.php
public function loadExtension(
    array $config,
    ContainerConfigurator $containerConfigurator,
    ContainerBuilder $containerBuilder,
): void {
    $containerConfigurator->import('../config/services.yaml');
}
```

Then enable autowiring/autoconfiguration and register `src/` as services,
excluding the classes that must not be services:

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    Somevendor\ContaoExampleBundle\:
        resource: ../src
        exclude: ../src/{ContaoManager,DependencyInjection,ContaoExampleBundle.php}
```

`autoconfigure: true` is what lets PHP attributes tag content elements, front end
modules, hooks, crons, and DCA callbacks automatically. Note Symfony recommends
explicit service definitions for public/reusable bundles; tighten this if needed.
Routes are not auto-loaded for a bundle either — register them via
`RoutingPluginInterface` (see the references file). Alternatives such as a
DependencyInjection `Extension` class are in
[references/manager-plugin-interfaces.md](references/manager-plugin-interfaces.md).

## Publishing essentials

Choose the right composer `type`:

- **`contao-bundle`** — couples to Contao and provides user-facing features; this
  is what makes it discoverable in the Contao Manager.
- **`symfony-bundle`** — couples to Symfony but not Contao specifically.
- **`library`** (or `contao-library`) — helpers/services for other developers,
  no user features.
- `contao-module` is legacy (Contao 3) and discouraged.

To appear in the Contao Manager / `extensions.contao.org` search, all must hold:
published on Packagist, `type` is `contao-bundle`, at least one **version tag**
(branch-only packages are ignored), and the `extra.contao-manager-plugin`
reference is present. Tag a release, push to a public Git repo, submit to
Packagist. Enrich the listing via the `contao/package-metadata` repository.

For private/commercial distribution (artifacts, `contao-provider` packages) and
the local `path`-repository dev loop, see
[references/local-development.md](references/local-development.md).
