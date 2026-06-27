# phpunit.xml.dist for a Contao extension

A full, commented configuration with separate `unit` and `functional` suites, a
`<source>` section for coverage, and the two PHPUnit extensions shipped by
`contao/test-case`. Modelled on Contao's own `phpunit.xml.dist` but scoped to a
single extension. Requires PHPUnit `^12` (the schema version must match your
installed PHPUnit, e.g. `12.4`).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/12.4/phpunit.xsd"
    bootstrap="vendor/autoload.php"
    cacheDirectory=".phpunit.cache"
    defaultTestSuite="unit"
    colors="true"
    displayDetailsOnTestsThatTriggerDeprecations="true"
    displayDetailsOnTestsThatTriggerErrors="true"
    displayDetailsOnTestsThatTriggerNotices="true"
    displayDetailsOnTestsThatTriggerWarnings="true"
>
  <!-- Files eligible for code coverage. Entities and generated migrations
       are usually excluded from coverage requirements. -->
  <source>
    <include>
      <directory>./src</directory>
    </include>
    <exclude>
      <directory>./src/Entity</directory>
    </exclude>
  </source>

  <php>
    <env name="APP_ENV" value="test"/>
    <env name="APP_DEBUG" value=""/>
    <!-- Functional tests only: the kernel to boot and the test database -->
    <env name="KERNEL_CLASS" value="Somevendor\ContaoExampleBundle\Tests\Functional\app\AppKernel"/>
    <env name="DATABASE_URL" value="mysql://root@localhost:3306/contao_test"/>
  </php>

  <testsuites>
    <!-- Fast, isolated tests — no kernel, no database -->
    <testsuite name="unit">
      <directory>./tests</directory>
      <exclude>./tests/Fixtures</exclude>
      <exclude>./tests/Functional</exclude>
    </testsuite>

    <!-- Tests that boot the kernel and hit the database -->
    <testsuite name="functional">
      <directory>./tests/Functional</directory>
      <exclude>./tests/Functional/app</exclude>
    </testsuite>
  </testsuites>

  <!-- Provided by contao/test-case:
       - ClearCachePhpunitExtension clears the cache before/after the run
       - WarnXdebugPhpunitExtension warns if Xdebug is enabled (slows tests) -->
  <extensions>
    <bootstrap class="Contao\TestCase\ClearCachePhpunitExtension"/>
    <bootstrap class="Contao\TestCase\WarnXdebugPhpunitExtension"/>
  </extensions>
</phpunit>
```

## Composer scripts

```json
{
    "require-dev": {
        "contao/test-case": "^5.0 || ^6.0",
        "phpunit/phpunit": "^12.4"
    },
    "scripts": {
        "unit-tests": "@php vendor/bin/phpunit --testsuite=unit",
        "functional-tests": "@php vendor/bin/phpunit --testsuite=functional"
    }
}
```

Run with `composer unit-tests` / `composer functional-tests`, or call
`vendor/bin/phpunit --testsuite=unit` directly. Pin the `contao/test-case`
constraint to the Contao major version you target.

## Notes

- The schema URL version (`12.4`) must line up with the PHPUnit version you
  install; bump both together.
- `defaultTestSuite="unit"` makes a bare `phpunit` run only the fast suite.
- Excluding `Fixtures` and `Functional` from the `unit` suite keeps unit runs
  free of kernel/database dependencies.
- The `<source>` section is optional but required if you want coverage scoping.
