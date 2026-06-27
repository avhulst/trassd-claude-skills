---
name: contao-testing
description: >-
  Test a Contao extension with PHPUnit — unit tests with ContaoTestCase
  (framework/adapter mocks, a configured DI container, a temp dir), functional
  tests with FunctionalTestCase (YAML fixtures, database schema reset), the
  ContaoDatabaseTrait, and a phpunit.xml.dist with separate unit/functional
  suites. Use when adding tests under tests/, configuring PHPUnit for a Contao
  bundle, or testing controllers, services, callbacks, or models.
---

# Contao Testing

Contao ships a dedicated PHPUnit helper library, **`contao/test-case`**, that an
extension uses to write unit and functional tests. It is a normal public
package — require it as a dev dependency and extend its base classes.

```bash
composer require --dev contao/test-case
```

It provides two base classes plus a database trait:

- `Contao\TestCase\ContaoTestCase` — for **unit tests**. Mocks the Contao
  framework and its adapters, builds a DI container with the Contao core
  configuration, and manages a per-class temp directory.
- `Contao\TestCase\FunctionalTestCase` — for **functional tests**. Extends
  Symfony's `WebTestCase`; boots the kernel, loads YAML fixtures, and resets the
  database schema between tests.
- `Contao\TestCase\ContaoDatabaseTrait` — a lower-level DBAL connection helper
  for loading a raw SQL file into a real test database.

Put tests under `tests/` and run them through Composer scripts (see below).

## Unit tests with ContaoTestCase

Extend `ContaoTestCase` and use its protected helpers — never hand-roll a
`ContaoFramework` mock. All helper names below exist verbatim in the source.

```php
namespace Somevendor\ContaoExampleBundle\Tests;

use Contao\Config;
use Contao\TestCase\ContaoTestCase;

class ExampleServiceTest extends ContaoTestCase
{
    public function testReadsConfig(): void
    {
        $framework = $this->createContaoFrameworkStub();
        $config = $framework->getAdapter(Config::class);

        self::assertTrue($config->isComplete());
    }
}
```

Key helpers (verified against `test-case/src/ContaoTestCase.php`):

- **Framework** — `createContaoFrameworkMock(array $adapters = [], array $instances = [])`
  and `createContaoFrameworkStub(...)`. A `Config` adapter with the default
  Contao configuration is added automatically when you don't supply one.
- **Adapters** — `createAdapterMock(array $methods)` / `createAdapterStub(array $methods)`
  create an adapter exposing the named methods; `createConfiguredAdapterMock(array $config)`
  / `createConfiguredAdapterStub(array $config)` take a `['method' => $returnValue]`
  map for the common stub-and-return case. Pass adapters into the framework via
  `[Contao\FilesModel::class => $adapter]`.
- **Magic-property classes** (models, `Database\Result`, etc.) —
  `createClassWithPropertiesMock(string $class, array $properties = [])` /
  `createClassWithPropertiesStub(...)` build an object whose `__get`/`__set`
  honor the given properties.
- **DI container** — `getContainerWithContaoConfiguration(string $projectDir = '')`
  returns a Symfony `ContainerBuilder` preloaded with the Contao core extension
  (parameters like `contao.upload_path`, `kernel.project_dir`, etc.).
- **Security** — `createTokenStorageStub(string $class)` returns a
  `TokenStorageInterface` stub whose token yields a back-end or front-end user
  (`$class` must be a `Contao\User` subclass).
- **Temp dir** — `getTempDir()` returns a per-class temp directory, auto-removed
  in `tearDownAfterClass()`.

> If you override `tearDownAfterClass()`, you **must** call
> `parent::tearDownAfterClass()`, or the temp directory leaks. Likewise call
> `parent::tearDown()` if you override `tearDown()` (the base unsets
> `$GLOBALS['TL_CONFIG']`).

The deprecated `mock*` aliases (`mockContaoFramework`, `mockAdapter`,
`mockConfiguredAdapter`, `mockClassWithProperties`, `mockTokenStorage`) still
exist but emit deprecations — always use the `create*` forms.

A full sample unit test is in
[references/unit-test.md](references/unit-test.md).

## Functional tests with FunctionalTestCase

Extend `FunctionalTestCase` (a Symfony `WebTestCase`) when you need a booted
kernel, a real HTTP client, and a real database. Boot the kernel first, then
load fixtures.

```php
namespace Somevendor\ContaoExampleBundle\Tests\Functional;

use Contao\TestCase\FunctionalTestCase;

class ExampleControllerTest extends FunctionalTestCase
{
    public function testRendersPage(): void
    {
        $client = self::createClient();
        self::loadFixtures([__DIR__.'/fixtures/page.yaml']);

        $client->request('GET', '/');
        self::assertResponseIsSuccessful();
    }
}
```

- `loadFixtures(array $yamlFiles)` (static) resets the schema, then inserts each
  YAML file's rows. The kernel must already be booted (call `self::createClient()`
  or `self::bootKernel()` first) — it throws otherwise.
- `resetDatabaseSchema()` (static) drops/recreates tables from Doctrine ORM
  metadata and truncates populated tables; `loadFixtures()` calls it for you.
- Fixture YAML is keyed by table name, each value a list of row maps. A special
  `sql` key runs raw statements:

  ```yaml
  tl_page:
      - { id: 1, pid: 0, type: root, dns: '', language: en }
      - { id: 2, pid: 1, type: regular, alias: index }
  sql:
      - "UPDATE tl_page SET published = 1"
  ```

Set `KERNEL_CLASS` and `DATABASE_URL` in `phpunit.xml.dist` (see below). A full
sample functional test plus fixture file is in
[references/functional-test.md](references/functional-test.md).

### ContaoDatabaseTrait

For tests that talk to a database directly (no kernel), `use ContaoDatabaseTrait`
to get `getConnection()` (a Doctrine DBAL `Connection`) and
`loadFileIntoDatabase(string $sqlFile)` to import a `.sql` dump. The connection
is built from environment variables: `DATABASE_URL`, or the discrete
`DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASS` / `DB_NAME`.

## phpunit.xml.dist

Define two test suites — `unit` and `functional` — bootstrap the Composer
autoloader, and register the two PHPUnit extensions shipped by `contao/test-case`.
`ClearCachePhpunitExtension` clears the cache around the run;
`WarnXdebugPhpunitExtension` warns if Xdebug is on (it slows tests down).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/12.4/phpunit.xsd"
    bootstrap="vendor/autoload.php"
    cacheDirectory=".phpunit.cache"
    defaultTestSuite="unit"
    colors="true"
>
  <php>
    <env name="APP_ENV" value="test"/>
    <env name="KERNEL_CLASS" value="Somevendor\ContaoExampleBundle\Tests\Functional\app\AppKernel"/>
    <env name="DATABASE_URL" value="mysql://root@localhost:3306/contao_test"/>
  </php>
  <testsuites>
    <testsuite name="unit">
      <directory>./tests</directory>
      <exclude>./tests/Functional</exclude>
    </testsuite>
    <testsuite name="functional">
      <directory>./tests/Functional</directory>
    </testsuite>
  </testsuites>
  <extensions>
    <bootstrap class="Contao\TestCase\ClearCachePhpunitExtension"/>
    <bootstrap class="Contao\TestCase\WarnXdebugPhpunitExtension"/>
  </extensions>
</phpunit>
```

A commented full `phpunit.xml.dist` (with a `<source>` section for coverage) is
in [references/phpunit-config.md](references/phpunit-config.md).

## Running the tests

Wire up Composer scripts so contributors run a single command:

```json
{
    "scripts": {
        "unit-tests": "@php vendor/bin/phpunit --testsuite=unit",
        "functional-tests": "@php vendor/bin/phpunit --testsuite=functional"
    }
}
```

```bash
composer unit-tests
composer functional-tests
# or directly
vendor/bin/phpunit --testsuite=unit
```

Functional tests need a reachable MySQL database matching `DATABASE_URL`; unit
tests do not.

## Rules

- Require `contao/test-case` as a **dev** dependency only.
- Unit tests extend `ContaoTestCase`; functional tests extend
  `FunctionalTestCase`. Don't mix HTTP/database concerns into a unit test.
- Mock the framework and adapters with the `create*` helpers — not the
  deprecated `mock*` aliases, and not raw PHPUnit `createMock(ContaoFramework::class)`.
- Boot the kernel before `loadFixtures()`/`resetDatabaseSchema()`.
- Always call the parent `tearDown()` / `tearDownAfterClass()` when overriding.
- Keep `unit` and `functional` as separate suites; exclude `Functional` and
  fixtures from the unit suite.

## Checklist

- [ ] `contao/test-case` in `require-dev`.
- [ ] Unit tests under `tests/` extend `ContaoTestCase` and use `create*` helpers.
- [ ] Functional tests under `tests/Functional/` extend `FunctionalTestCase`,
      boot the kernel, and load YAML fixtures.
- [ ] `phpunit.xml.dist` defines `unit` + `functional` suites, the autoload
      bootstrap, and the two `contao/test-case` PHPUnit extensions.
- [ ] `KERNEL_CLASS` and `DATABASE_URL` set for functional tests.
- [ ] `unit-tests` / `functional-tests` Composer scripts exist.
