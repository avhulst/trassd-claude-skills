# Advanced migration control

By default, installing or updating a plugin runs all (new) migrations
automatically. If you need finer control over *which* migrations run during a
lifecycle event, a plugin must first reject the automatic execution and then
drive the collection itself.

The plugin-scoped `MigrationCollection` is available from the `InstallContext`
and its subclasses (`UpdateContext`, `ActivateContext`, …).

```php
public function update(UpdateContext $updateContext): void
{
    // disable automatic migration execution for this lifecycle step
    $updateContext->setAutoMigrate(false);

    $migrationCollection = $updateContext->getMigrationCollection();

    // run all DESTRUCTIVE migrations up to and including a given timestamp
    $migrationCollection->migrateDestructiveInPlace(1572566400);

    // run all UPDATE migrations up to and including a given timestamp
    $migrationCollection->migrateInPlace(1576143014);
}
```

Notes:

- If the plugin does not use the Shopware migration system, an empty
  (NullObject) collection is provided in the context.
- The timestamps passed to `migrateInPlace()` / `migrateDestructiveInPlace()`
  are the same Unix timestamps used in migration class names / returned by
  `getCreationTimestamp()`.

## Major-version execution order (core context)

Core migrations are grouped by major version namespace (e.g. `Core\Migration\V6_5`)
and run oldest-major first, then legacy migrations. You can target a single major
version on the CLI:

```bash
./bin/console database:migrate --all core.V6_7
```

Destructive execution offers selection modes for customers (`mode=all`,
`mode=blue-green`, `mode=safe`); `mode=safe` is the default. Plugin migrations
should keep minor/patch changes non-destructive so they remain compatible with
blue-green deployment.
