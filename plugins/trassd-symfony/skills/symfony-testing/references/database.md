# Database testing & smoke tests

## Configure a separate test database

Use a dedicated database so tests never touch other environments. Set the URL
in `.env.test.local` (machine-specific) or `.env.test` (shared, if identical
everywhere). A common convention is the `_test` suffix (`project_acme` →
`project_acme_test`).

```env
# .env.test.local
DATABASE_URL="mysql://USERNAME:PASSWORD@127.0.0.1:3306/DB_NAME?serverVersion=8.0.37"
```

Create it:

```terminal
php bin/console --env=test doctrine:database:create
php bin/console --env=test doctrine:schema:create
```

## Reset the database between tests

Tests must be independent. Install `dama/doctrine-test-bundle`:

```terminal
composer require --dev dama/doctrine-test-bundle
```

Enable it as a PHPUnit extension; it begins a transaction before every test and
rolls it back afterward to undo all changes:

```xml
<!-- phpunit.dist.xml -->
<phpunit>
    <extensions>
        <!-- PHPUnit 10+ -->
        <bootstrap class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension"/>
        <!-- PHPUnit < 10 -->
        <extension class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension"/>
    </extensions>
</phpunit>
```

## Load fixtures

```terminal
composer require --dev doctrine/doctrine-fixtures-bundle
php bin/console make:fixtures
```

```php
// src/DataFixtures/ProductFixture.php
namespace App\DataFixtures;

use App\Entity\Product;
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Persistence\ObjectManager;

class ProductFixture extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        $product = new Product();
        $product->setName('Priceless widget');
        $product->setPrice(14.50);
        $manager->persist($product);

        $manager->flush();
    }
}
```

Load them (empties the DB and reloads all fixture classes):

```terminal
php bin/console --env=test doctrine:fixtures:load
```

## Test a repository against a real database

Do **not** mock repositories in unit tests — test them functionally. Get the
entity manager from the container and close it in `tearDown()` to avoid memory
leaks:

```php
// tests/Repository/ProductRepositoryTest.php
namespace App\Tests\Repository;

use App\Entity\Product;
use Doctrine\ORM\EntityManager;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class ProductRepositoryTest extends KernelTestCase
{
    private ?EntityManager $entityManager;

    protected function setUp(): void
    {
        $kernel = self::bootKernel();
        $this->entityManager = $kernel->getContainer()
            ->get('doctrine')
            ->getManager();
    }

    public function testSearchByName(): void
    {
        $product = $this->entityManager
            ->getRepository(Product::class)
            ->findOneBy(['name' => 'Priceless widget']);

        $this->assertSame(14.50, $product->getPrice());
    }

    protected function tearDown(): void
    {
        parent::tearDown();

        $this->entityManager->close();
        $this->entityManager = null;
    }
}
```

## Smoke-test every URL

A single data-provider test that requests every URL and asserts success catches
broad breakage cheaply. Add it when you create the app; layer specific tests
on top later.

```php
// tests/ApplicationAvailabilityFunctionalTest.php
namespace App\Tests;

use PHPUnit\Framework\Attributes\DataProvider;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class ApplicationAvailabilityFunctionalTest extends WebTestCase
{
    #[DataProvider('urlProvider')]
    public function testPageIsSuccessful($url): void
    {
        $client = self::createClient();
        $client->request('GET', $url);

        $this->assertResponseIsSuccessful();
    }

    public static function urlProvider(): \Generator
    {
        yield ['/'];
        yield ['/posts'];
        yield ['/post/fixture-post-1'];
        // ...
    }
}
```
