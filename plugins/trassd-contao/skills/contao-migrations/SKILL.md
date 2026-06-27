---
name: contao-migrations
description: >-
  Write Contao database migrations — the MigrationInterface / AbstractMigration,
  the contao.migration service tag, shouldRun()/run() returning a MigrationResult,
  and running them with the contao:migrate command. Use when adding a migration
  under a bundle's or app's Migration/ directory, when the DCA schema diff cannot
  express a needed change (data transforms, renames, structural fixes), or when
  asked how Contao executes migrations during an update.
---

# Contao Migrations

Contao runs migrations during the install-tool database update and via the
`contao:migrate` console command. Use this skill when writing a migration
service or reasoning about when one is needed.

## Migrations vs. the DCA SQL schema

Contao manages two distinct mechanisms — don't conflate them:

- **DCA `sql` schema** — column/table definitions declared in the DCA are
  applied automatically by the install tool / `contao:migrate` as a schema
  diff. Add or change a column by editing the DCA, not by writing a migration.
- **Migrations** — services that perform changes the schema diff *cannot*
  express on its own: transforming existing data, renaming or merging columns,
  backfilling values, or other structural fixups tied to an update.

Rule of thumb: if a plain schema diff would lose or mangle data (e.g. merging
`firstName` + `lastName` into `name`), write a migration to carry the data
across, then let the schema diff drop the obsolete columns.

## Defining a migration

Create a service that implements `Contao\CoreBundle\Migration\MigrationInterface`
and tag it `contao.migration`. With autoconfiguration enabled (the default in a
Contao app/bundle), tagging is automatic — just place the class under a
`Migration/` directory. The interface requires three methods:

- **`getName(): string`** — human-readable description shown to the user when
  they are asked whether to run the migration.
- **`shouldRun(): bool`** — checks prerequisites and whether the migration is
  still needed. Write this *very defensively*: the database may be empty or in
  an unexpected state when it is called (check that tables/columns exist before
  inspecting them).
- **`run(): MigrationResult`** — performs the migration; only called when
  `shouldRun()` returned `true`. Returns a `MigrationResult` (success flag +
  message). To abort the whole migration process on an unrecoverable error,
  throw an exception.

### Prefer AbstractMigration

Extend `Contao\CoreBundle\Migration\AbstractMigration` instead of implementing
the interface directly. It provides:

- a default `getName()` returning the class FQCN (override it for a friendlier
  description);
- `createResult(bool $successful, ?string $message = null)`, which builds a
  `MigrationResult` with a sensible default message.

You still implement `shouldRun()` and `run()` yourself.

```php
namespace App\Migration;

use Contao\CoreBundle\Migration\AbstractMigration;
use Contao\CoreBundle\Migration\MigrationResult;
use Doctrine\DBAL\Connection;

class CustomerNameMigration extends AbstractMigration
{
    public function __construct(private readonly Connection $connection)
    {
    }

    public function shouldRun(): bool
    {
        $schemaManager = $this->connection->createSchemaManager();

        // Defensive: bail out if the table is not present yet.
        if (!$schemaManager->tablesExist(['tl_customers'])) {
            return false;
        }

        $columns = $schemaManager->listTableColumns('tl_customers');

        return isset($columns['firstname'], $columns['lastname'])
            && !isset($columns['name']);
    }

    public function run(): MigrationResult
    {
        // ... ALTER / UPDATE statements via $this->connection ...

        return $this->createResult(true, 'Combined customer names.');
    }
}
```

See [references/example-migration.md](references/example-migration.md) for the
full data-merge example with the actual SQL statements.

## Registration

Autoconfiguration tags any `MigrationInterface` service with `contao.migration`
automatically. To register or prioritise explicitly:

```yaml
# config/services.yaml
services:
    App\Migration\CustomerNameMigration:
        tags:
            - { name: contao.migration, priority: 0 }
```

Higher `priority` runs earlier. Keep migrations ordering-independent where
possible by guarding everything through `shouldRun()`.

## Running migrations

```
vendor/bin/contao-console contao:migrate
```

This applies the DCA schema diff and runs every pending migration whose
`shouldRun()` returns `true`. The same process runs from the install tool's
database update. Migrations are also executed automatically as part of the
Contao/extension update flow, so a migration written today will run on
downstream installs during their next update.

## Checklist

- [ ] Change genuinely needs a migration (not just a DCA `sql` edit).
- [ ] Class lives under a `Migration/` directory; extends `AbstractMigration`.
- [ ] `shouldRun()` checks table/column existence before touching them and is
      idempotent (returns `false` once the migration has been applied).
- [ ] `run()` returns `createResult(...)` / a `MigrationResult`; throws only to
      abort the whole process.
- [ ] Verified with `vendor/bin/contao-console contao:migrate`.
