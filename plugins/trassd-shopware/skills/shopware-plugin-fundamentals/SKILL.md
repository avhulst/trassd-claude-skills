---
name: shopware-plugin-fundamentals
description: >-
  Scaffold and structure a Shopware 6 plugin — the plugin base class,
  composer.json, lifecycle methods (install/activate/update/uninstall), and
  plugin configuration. Triggers when creating a plugin, editing its base
  class, or adding config.xml. Use when you need the correct directory layout,
  the required composer.json fields (type, shopware/core require,
  extra.shopware-plugin-class, PSR-4 autoload), the lifecycle hooks and their
  context objects, the bin/console plugin commands, or how to define and read
  config via SystemConfigService.
---

# Shopware 6 Plugin Fundamentals

A Shopware plugin is a PHP server-side extension. Technically it is a Symfony
bundle with a Shopware lifecycle (install/update/deactivate/uninstall) and can
be managed from the Administration. Use a plugin for backend logic, DI services,
events, migrations, entities, commands, or Admin/Storefront extensions.

For project work, prefer **static plugins** in
`<project root>/custom/static-plugins` — committed to Git and required via
Composer. **Managed plugins** live in `custom/plugins` and are typically
installed via the Administration (used for marketplace distribution).

## Directory structure

Minimal installable layout:

```text
SwagBasicExample/
├── composer.json
└── src/
    └── SwagBasicExample.php
```

- Name the plugin in **UpperCamelCase** with a vendor prefix (e.g.
  `SwagBasicExample`). A vendor prefix is required to publish in the Community Store.
- Scaffold it with `bin/console plugin:create SwagBasicExample`
  (add `--create-config` to also generate a demo `config.xml`).
- The `src/` directory is recommended but not mandatory — if you change the
  `autoload.psr-4` path, your folder structure must match.

## composer.json (required fields)

Shopware reads `composer.json` to recognize and register the plugin. At minimum:

```json
{
    "name": "swag/basic-example",
    "version": "1.0.0",
    "type": "shopware-platform-plugin",
    "license": "MIT",
    "require": {
        "shopware/core": "~6.6.0"
    },
    "extra": {
        "shopware-plugin-class": "Swag\\BasicExample\\SwagBasicExample",
        "label": { "en-GB": "The displayed readable name for the plugin" }
    },
    "autoload": {
        "psr-4": { "Swag\\BasicExample\\": "src/" }
    }
}
```

Rules:
- `"type": "shopware-platform-plugin"` — required so Shopware recognizes it.
- `require` must include `shopware/core` (compatibility check).
- `extra.shopware-plugin-class` must reference the base PHP class FQCN.
- `autoload.psr-4` namespace must match the directory structure.
- The `extra.label` / `extra.description` are translatable (per locale) and
  shown in the Administration. A `version` warning from `plugin:refresh` is harmless.

## Plugin base class

The base class extends `Shopware\Core\Framework\Plugin` (which extends
`Shopware\Core\Framework\Bundle`, which extends the Symfony `Bundle`). It acts as
the bootstrap file and exposes the service container via `$this->container`.

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample;

use Shopware\Core\Framework\Plugin;

class SwagBasicExample extends Plugin
{
}
```

## Lifecycle methods

Override only the hooks you need on the base class. Each receives a context
object from `Shopware\Core\Framework\Plugin\Context\*`.

| Method | When it runs | Context |
| :--- | :--- | :--- |
| `install()` | on install | `InstallContext` |
| `postInstall()` | after successful install | `InstallContext` |
| `activate()` | before activation | `ActivateContext` |
| `deactivate()` | before deactivation | `DeactivateContext` |
| `update()` | on update | `UpdateContext` |
| `postUpdate()` | after successful update | `UpdateContext` |
| `uninstall()` | on uninstall | `UninstallContext` |

```php
public function install(InstallContext $installContext): void
{
    // Prepare requirements / create entities — but keep them inactive
}
```

Key rules:
- **Do not create active business data in `install()`.** The plugin is not yet
  active. Create entities here (e.g. a payment method) but keep them inactive;
  activate them in `activate()`.
- **`update()` should rarely touch the database.** Use idempotent
  [migrations](../shopware-database-migrations) for schema/data changes;
  reserve `update()` for feature toggles or version-dependent logic. The
  `UpdateContext` adds `getUpdatePluginVersion()` (target) and
  `getCurrentPluginVersion()` (installed).
- **`uninstall()` must respect `keepUserData()`.** If
  `$uninstallContext->keepUserData()` is `true`, return early and delete
  nothing. Never blindly delete entities referenced by production data (e.g. a
  payment method used in real orders) — deactivate instead.
- `InstallContext` provides the current plugin/Shopware versions, the system
  `Context`, and auto-migration control (`isAutoMigrate()` / `setAutoMigrate()`).

See [references/lifecycle.md](references/lifecycle.md) for full method bodies
including the `keepUserData()` guard.

## Install / activate via CLI

```bash
bin/console plugin:refresh                          # detect plugins, list status
bin/console plugin:install --activate SwagBasicExample
```

`plugin:refresh` makes Shopware recognize the plugin (it appears in the list as
not installed). `plugin:install --activate` installs and activates in one step.

## Plugin configuration (config.xml)

Define a configuration page with `src/Resources/config/config.xml` — no custom
Admin module needed; it renders automatically.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/shopware/trunk/src/Core/System/SystemConfig/Schema/config.xsd">
    <card>
        <title>Minimal configuration</title>
        <input-field>
            <name>example</name>
        </input-field>
    </card>
</config>
```

Rules:
- At least one `<card>` with a `<title>` and one `<input-field>`.
- Every `<input-field>` starts with `<name>` — unique, non-translatable, min 4
  chars, pattern `[a-zA-Z][a-zA-Z0-9]*`.
- Field type via the `type` attribute (default `text`); options include
  `bool`, `int`, `float`, `single-select`, `password`, `colorpicker`, etc.
- `<title>`, `<label>`, `<placeholder>`, `<helpText>`, `<option><name>` are
  translatable via `lang` (default `en-GB`).
- `<defaultValue>` is imported into the database on install/update.

See [references/config-xml.md](references/config-xml.md) for input-field
settings, types, and entity-select components.

## Reading configuration

Plugin config keys are prefixed: `<BundleName>.config.<configName>`, e.g.
`SwagBasicExample.config.example`. This avoids collisions between plugins.

**PHP** — inject `Shopware\Core\System\SystemConfig\SystemConfigService` and call
`get()`. The second argument is the sales-channel ID (`null` = global / all
sales channels):

```php
$value = $this->systemConfigService->get('SwagBasicExample.config.example', $salesChannelId);
```

**Administration (JS):** use `systemConfigApiService.getValues('SwagBasicExample.config')`
(needs the `system_config:read` permission).

**Storefront (Twig):** `{{ config('SwagBasicExample.config.example') }}`.
