# Plugin lifecycle methods — full examples

The base plugin class (`src/SwagBasicExample.php`) extends
`Shopware\Core\Framework\Plugin`. Override only the hooks you need. The service
container is available as `$this->container`. Each context class lives under
`Shopware\Core\Framework\Plugin\Context\`.

## install

Runs on install. Register entities, create initial data, prepare requirements.
Keep created entities inactive until `activate()`.

```php
public function install(InstallContext $installContext): void
{
    // e.g. create a payment method but leave it inactive
}
```

`InstallContext` provides:
- current plugin version, current Shopware version
- the system `Context` (language, currency, permissions)
- migration access
- auto-migration control: `isAutoMigrate()` / `setAutoMigrate(bool)`

## activate / deactivate

```php
public function activate(ActivateContext $context): void
{
    // activate entities created during install, or create data
    // that could not be safely created while the plugin was inactive
}

public function deactivate(DeactivateContext $context): void
{
    // deactivate or remove entities that must not stay active
    // while the plugin is inactive
}
```

Both contexts expose the same information as `InstallContext`.

## update / postUpdate

Prefer idempotent migrations for database changes. Use `update()` for
non-database adjustments, feature toggles, or version-dependent logic.

```php
public function update(UpdateContext $context): void
{
    $from = $context->getCurrentPluginVersion(); // version before update
    $to   = $context->getUpdatePluginVersion();  // target version
    // version-dependent, non-database logic
}
```

## postInstall / postUpdate

Run after a successful install / update — for one-time actions that should only
happen once the process fully completes.

```php
public function postInstall(InstallContext $context): void {}
public function postUpdate(UpdateContext $context): void {}
```

## uninstall

Runs on uninstall. Always honor `keepUserData()` and never blindly delete
entities referenced by production data (deactivate instead).

```php
public function uninstall(UninstallContext $uninstallContext): void
{
    if ($uninstallContext->keepUserData()) {
        return;
    }

    // remove or deactivate the data created by the plugin
}
```
