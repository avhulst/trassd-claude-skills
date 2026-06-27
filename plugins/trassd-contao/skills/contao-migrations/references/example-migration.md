# Example: merging two columns into one

Scenario: a table `tl_customers` historically had `firstName` and `lastName`
columns. A new version combines them into a single `name` column. The DCA schema
diff would happily add `name` and later drop the old columns, but it cannot
carry the existing values across — so a migration backfills `name` before the
old columns disappear.

The migration only acts while the database is still in the *old* shape
(`firstName` + `lastName` present, `name` absent), which also makes it
idempotent: once `name` exists, `shouldRun()` returns `false`.

```php
// src/Migration/CustomerNameMigration.php
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

        // If the table itself does not exist there is nothing to migrate.
        if (!$schemaManager->tablesExist(['tl_customers'])) {
            return false;
        }

        $columns = $schemaManager->listTableColumns('tl_customers');

        return isset($columns['firstname'])
            && isset($columns['lastname'])
            && !isset($columns['name']);
    }

    public function run(): MigrationResult
    {
        $this->connection->executeQuery("
            ALTER TABLE
                tl_customers
            ADD
                name varchar(255) NOT NULL DEFAULT ''
        ");

        $stmt = $this->connection->prepare("
            UPDATE
                tl_customers
            SET
                name = CONCAT(firstName, ' ', lastName)
        ");

        $stmt->execute();

        return $this->createResult(
            true,
            'Combined '.$stmt->rowCount().' customer names.'
        );
    }
}
```

Notes:

- `MigrationResult` carries a success flag and a message
  (`isSuccessful()` / `getMessage()`); build it through `createResult()` from
  `AbstractMigration`.
- The DBAL `Connection` is injected; `createSchemaManager()` is the modern DBAL
  accessor used in `shouldRun()` for existence checks.
- Use a `name`/priority on the `contao.migration` tag only if ordering matters;
  otherwise rely on autoconfiguration and `shouldRun()` guards.
