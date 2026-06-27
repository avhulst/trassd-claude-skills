# Integration tests (KernelTestCase)

## Boot the kernel and fetch a service

```php
// tests/Service/NewsletterGeneratorTest.php
namespace App\Tests\Service;

use App\Service\NewsletterGenerator;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class NewsletterGeneratorTest extends KernelTestCase
{
    public function testSomething(): void
    {
        // (1) boot the Symfony kernel
        self::bootKernel();

        // (2) access the special test service container
        $container = static::getContainer();

        // (3) run the service & assert
        $generator = $container->get(NewsletterGenerator::class);
        $newsletter = $generator->generateMonthlyNews(/* ... */);

        $this->assertEquals('...', $newsletter->getContent());
    }
}
```

The container returned by `static::getContainer()` is a test-only container
that exposes both public services and non-removed private services. The kernel
is rebooted for each test, isolating them. The kernel class is read from the
`KERNEL_CLASS` env var:

```env
# .env.test
KERNEL_CLASS=App\Kernel
```

If a private service has been removed (unused by anything else), declare it
public in `config/services_test.yaml` to test it.

## Booting with different options

```php
self::bootKernel([
    'environment' => 'my_test_env',
    'debug'       => false,
]);
```

Running with `debug => false` on CI speeds tests up (no cache clearing), but
then you must ensure a clean cache yourself, e.g. in `tests/bootstrap.php`:

```php
(new \Symfony\Component\Filesystem\Filesystem())->remove(__DIR__.'/../var/cache/test');
```

## Mocking a dependency

Set a mock on the test container *before* fetching the service under test; the
mock is injected automatically (works for private services and aliases too):

```php
use App\Contracts\Repository\NewsRepositoryInterface;

class NewsletterGeneratorTest extends KernelTestCase
{
    public function testSomething(): void
    {
        self::bootKernel();
        $container = static::getContainer();

        $newsRepository = $this->createMock(NewsRepositoryInterface::class);
        $newsRepository->expects(self::once())
            ->method('findNewsFromLastMonth')
            ->willReturn([
                new News('some news'),
                new News('some other news'),
            ]);

        $container->set(NewsRepositoryInterface::class, $newsRepository);

        // injected with the mocked repository
        $generator = $container->get(NewsletterGenerator::class);
        // ...
    }
}
```
