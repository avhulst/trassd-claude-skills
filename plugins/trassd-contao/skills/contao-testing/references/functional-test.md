# Sample functional test (FunctionalTestCase)

Functional tests boot a real kernel and use a real MySQL database. They extend
`Contao\TestCase\FunctionalTestCase`, which extends Symfony's
`Symfony\Bundle\FrameworkBundle\Test\WebTestCase`.

```php
<?php

declare(strict_types=1);

namespace Somevendor\ContaoExampleBundle\Tests\Functional;

use Contao\TestCase\FunctionalTestCase;

class ExampleControllerTest extends FunctionalTestCase
{
    public function testTheHomePageIsRendered(): void
    {
        // 1. Boot the kernel and get an HTTP client
        $client = self::createClient();

        // 2. Load fixtures (this also resets the schema). Kernel must be booted.
        self::loadFixtures([
            __DIR__.'/fixtures/page.yaml',
        ]);

        // 3. Make a request and assert on the response
        $crawler = $client->request('GET', '/');

        self::assertResponseIsSuccessful();
        self::assertSelectorTextContains('h1', 'Welcome');
    }

    public function testTheDatabaseIsResetBetweenTests(): void
    {
        self::bootKernel();
        self::resetDatabaseSchema(); // drop/recreate tables from ORM metadata

        // ... start from a clean schema ...
    }
}
```

## Fixture file format

`loadFixtures()` parses each YAML file. Top-level keys are table names; each
value is a list of rows. Column values are inserted with quoted identifiers. A
special `sql` key runs raw statements instead of inserts.

```yaml
# tests/Functional/fixtures/page.yaml
tl_page:
    - { id: 1, pid: 0, sorting: 128, type: root, dns: '', language: en, fallback: 1 }
    - { id: 2, pid: 1, sorting: 128, type: regular, alias: index, title: Home }

tl_article:
    - { id: 1, pid: 2, sorting: 128, inColumn: main, title: Main }

sql:
    - "UPDATE tl_page SET published = 1"
```

## Required environment

Set these in `phpunit.xml.dist` (see `references/phpunit-config.md`):

- `KERNEL_CLASS` — the test `AppKernel` class to boot.
- `DATABASE_URL` — DSN of a reachable MySQL test database, e.g.
  `mysql://root@localhost:3306/contao_test`.
- `APP_ENV=test`.

## Behavior notes (from FunctionalTestCase)

- `loadFixtures()` and `resetDatabaseSchema()` are **static** and require a booted
  kernel; they throw a `RuntimeException` otherwise.
- The first `resetDatabaseSchema()` creates the schema from Doctrine ORM metadata
  via `SchemaTool`; subsequent resets truncate populated tables and only rebuild
  a table when its columns changed — so repeated runs are fast.
- The base `tearDown()` restores the default exception handler; call
  `parent::tearDown()` if you override it.
