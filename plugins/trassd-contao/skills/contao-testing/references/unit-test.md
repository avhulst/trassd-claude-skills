# Sample unit test (ContaoTestCase)

A complete unit test for a service that reads a model through the Contao
framework. All helper methods come from `Contao\TestCase\ContaoTestCase`.

```php
<?php

declare(strict_types=1);

namespace Somevendor\ContaoExampleBundle\Tests\Service;

use Contao\Config;
use Contao\PageModel;
use Contao\TestCase\ContaoTestCase;
use Somevendor\ContaoExampleBundle\Service\PageTitleResolver;

class PageTitleResolverTest extends ContaoTestCase
{
    public function testReturnsTheModelTitle(): void
    {
        // A magic-property model stub (PageModel uses __get/__set)
        $page = $this->createClassWithPropertiesStub(PageModel::class, [
            'id' => 2,
            'title' => 'Home',
        ]);

        // An adapter whose findByPk() returns the model — configured in one line
        $adapter = $this->createConfiguredAdapterStub([
            'findByPk' => $page,
        ]);

        // Wire the adapter into a framework stub (Config adapter is added for free)
        $framework = $this->createContaoFrameworkStub([
            PageModel::class => $adapter,
        ]);

        $resolver = new PageTitleResolver($framework);

        self::assertSame('Home', $resolver->getTitle(2));
    }

    public function testUsesTheDefaultContaoConfiguration(): void
    {
        $framework = $this->createContaoFrameworkStub();
        $config = $framework->getAdapter(Config::class);

        // The default Config adapter is always present and isComplete()
        self::assertTrue($config->isComplete());
    }

    public function testCanResolveServicesFromTheContainer(): void
    {
        $container = $this->getContainerWithContaoConfiguration();

        self::assertSame('files', $container->getParameter('contao.upload_path'));
    }
}
```

## Notes

- `createClassWithPropertiesStub()` / `createClassWithPropertiesMock()` are for
  classes with magic `__get`/`__set` (Contao models, `Database\Result`). For a
  read-only target pass the properties as the second argument.
- `createConfiguredAdapterStub(['method' => $value])` is shorthand for
  `createAdapterStub(['method'])` followed by `->method('method')->willReturn($value)`.
- A `Config` adapter is injected automatically by `createContaoFrameworkStub()` /
  `createContaoFrameworkMock()` unless you pass your own under `Config::class`.
- Use a **mock** (`create*Mock`) when you need to assert call expectations
  (`->expects($this->once())`), and a **stub** (`create*Stub`) when you only need
  canned return values.

## Temp directory and teardown

If your test writes files, use `getTempDir()`; it is cleaned up automatically.
Only override teardown hooks if you must — and always call the parent.

```php
class CacheWriterTest extends ContaoTestCase
{
    public function testWritesToTheTempDir(): void
    {
        $fs = new \Symfony\Component\Filesystem\Filesystem();
        $fs->mkdir($this->getTempDir().'/var/cache');

        // ... assert against files under $this->getTempDir() ...
    }

    public static function tearDownAfterClass(): void
    {
        // Without this call the temporary directory would not be removed.
        parent::tearDownAfterClass();
    }
}
```
