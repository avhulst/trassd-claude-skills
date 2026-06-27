---
name: shopware-migrations
description: >-
  Write Shopware 6 database migrations — the MigrationStep base class,
  naming/timestamp conventions, update() vs updateDestructive(), and applying
  migrations. Triggers when adding a migration under a plugin's Migration/
  directory or otherwise changing the database schema in a Shopware plugin.
---

# Shopware Database Migrations

Migrations are PHP classes that apply incremental database schema changes. They
run automatically on plugin install/update and can be triggered manually during
development. Use migrations for schema and structural changes (not custom fields,
which extend entities without touching the schema).

## File location & naming

- Migrations live in `<plugin root>/src/Migration/` (relative to the plugin base
  class). Override the location by returning a different namespace from the
  plugin base class `getMigrationNamespace()` — the directory must then match the
  last namespace segment.
- File/class name pattern: `Migration<timestamp><Description>`, e.g.
  `Migration1611740369ExampleDescription`. The parts are:
  - `Migration` — required prefix.
  - `<timestamp>` — a Unix timestamp that makes migrations incremental/ordered.
  - `<Description>` — a descriptive name.
- The class extends `Shopware\Core\Framework\Migration\MigrationStep` and must
  return the same timestamp from `getCreationTimestamp()` that appears in the
  name. Never edit `getCreationTimestamp()` by hand.

## Generating the skeleton

Run from the Shopware root directory:

```bash
# Boilerplate migration for a plugin
./bin/console database:create-migration -p SwagBasicExample --name ExampleDescription

# Full migration with SQL generated from entity definitions (plugin must be active)
./bin/console dal:migration:create --bundle=SwagBasicExample --entities=your_entity,your_other_entity
```

## The two steps: update() vs updateDestructive()

A `MigrationStep` has three methods:

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Migration;

use Doctrine\DBAL\Connection;
use Shopware\Core\Framework\Migration\MigrationStep;

class Migration1611740369ExampleDescription extends MigrationStep
{
    public function getCreationTimestamp(): int
    {
        return 1611740369;
    }

    public function update(Connection $connection): void
    {
        // Non-destructive, backward-compatible, reversible changes.
        // e.g. CREATE TABLE IF NOT EXISTS ...
    }

    public function updateDestructive(Connection $connection): void
    {
        // Non-reversible changes (DROP COLUMN/TABLE). Runs after update(),
        // only when explicitly requested.
    }
}
```

- `update()` — backward-compatible, **reversible** changes (creating tables,
  adding columns). Runs automatically.
- `updateDestructive()` — **non-reversible** changes (dropping columns/tables).
  Runs *after* `update()` and is only executed explicitly, not automatically.
- Do **not** use `updateDestructive()` to revert `update()`. Reverting belongs in
  the plugin's `uninstall()` lifecycle method, not in the migration.

Use the **expand & contract** pattern for backward-compatible schema evolution:
add the new structure in `update()` (expand), migrate the data, and only drop the
old structure in `updateDestructive()` (contract) once the new code is verified.

## Executing migrations

On plugin install all migrations run; on plugin update only **new** migrations
run. To run manually during development:

```bash
# Run update() of pending migrations for a bundle
./bin/console database:migrate SwagBasicExample --all

# Run updateDestructive() of pending migrations
./bin/console database:migrate-destructive SwagBasicExample --all
```

The optional identifier argument selects which migrations run; it defaults to
Shopware Core. Pass the plugin's bundle name (e.g. `SwagBasicExample`) to run
that plugin's migrations.

For finer control during install/update, a lifecycle method may call
`$context->setAutoMigrate(false)` and then drive the `MigrationCollection`
manually via `migrateInPlace()` / `migrateDestructiveInPlace()`. See
[references/advanced-control.md](references/advanced-control.md).

## Core rules (must follow)

1. **Never change an already-executed/released migration.** Write a new one
   instead — otherwise updated systems diverge from fresh installs. (Only
   exception: a migration not yet in a public release.)
2. **Must be re-runnable (idempotent).** A migration can fail midway (timeout,
   connection, syntax) and be retried. Guard with `IF [NOT] EXISTS` on
   `CREATE`/`DROP`, and use the `MigrationStep` helpers `dropTableIfExists()` /
   `dropColumnIfExists()` and the `columnExists()` check. `ALTER TABLE` has no
   conditional check — query the schema yourself before altering.
3. **Don't trust identifiers.** IDs differ between customer and dev systems;
   query for them rather than hard-coding.
4. **Don't trust customer data.** Program defensively with exact queries; never
   assume rows or structures exist.
5. **Don't overwrite customized data.** A common guard is updating only rows
   where `updated_at IS NULL`.
6. **Performance.** A migration must never exceed ~10s on a local system; test
   against large data sets.
7. **No default language.** Don't assume en-GB/de-DE. Use `ImportTranslationsTrait`
   for translatable seed data.
8. **Write a migration test** for each migration; verify idempotence by running
   `update()` twice. DDL forces an implicit commit, so handle transactions
   manually in tests.
9. **Table naming.** Use descriptive, singular, snake_case names. Avoid the
   legacy `swag_` prefix.

Keep migrations free of business logic — they only perform schema/data changes.

See [references/examples.md](references/examples.md) for fuller migration and
helper-usage examples.
