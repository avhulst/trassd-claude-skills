# Migration examples

These examples are distilled from the Shopware database-migrations guide and the
core migration coding guidelines. Adapt names to your plugin/domain.

## Non-destructive: create a table in update()

`CREATE TABLE IF NOT EXISTS` keeps the migration re-runnable. Use descriptive,
singular, snake_case table names (no legacy `swag_` prefix).

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
        $query = <<<SQL
CREATE TABLE IF NOT EXISTS `example_general_settings` (
    `id`              INT          NOT NULL,
    `example_setting` VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
)
    ENGINE = InnoDB
    DEFAULT CHARSET = utf8mb4
    COLLATE = utf8mb4_unicode_ci;
SQL;

        $connection->executeStatement($query);
    }

    public function updateDestructive(Connection $connection): void
    {
        // intentionally empty
    }
}
```

## Destructive change deferred to updateDestructive()

Follow expand & contract: add the new column in `update()`, migrate the data,
and drop the old column only in `updateDestructive()` — never as a way to revert
`update()`.

```php
public function update(Connection $connection): void
{
    // expand: add the new column (ALTER TABLE has no IF EXISTS — guard manually
    // by querying the schema first if the migration may re-run)
    // migrate: copy data from the old column into the new one
}

public function updateDestructive(Connection $connection): void
{
    // contract: remove the obsolete column once the new code is verified
    $this->dropColumnIfExists($connection, 'example_general_settings', 'legacy_column');
}
```

`MigrationStep` provides `dropTableIfExists()` and `dropColumnIfExists()`; the
`columnExists()` helper (from the column-exists trait) lets you check before an
`ALTER TABLE`.

## Don't overwrite customized data

Only touch rows the customer has not modified — a typical guard is
`updated_at IS NULL`:

```sql
UPDATE `product` SET name = 'foobar' WHERE updated_at IS NULL;
```

## Translatable seed data

Don't assume a default language. Use the core `ImportTranslationsTrait`
(`Shopware\Core\Migration\Traits\ImportTranslationsTrait`) and supply
translations for every locale your data needs (e.g. de-DE and en-GB) rather than
hard-coding a single language.
