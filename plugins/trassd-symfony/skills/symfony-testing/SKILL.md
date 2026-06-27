---
name: symfony-testing
description: >-
  Write functional and integration tests the Symfony way — WebTestCase,
  KernelTestCase, the DOM crawler, and database testing. Triggers when adding or
  changing tests under tests/, or testing controllers/services.
---

# Symfony Testing

Symfony tests are PHPUnit tests living in `tests/`. Each test class ends with
`Test` (e.g. `BlogControllerTest`) and is run with `php bin/phpunit`. Install
the tooling once with `composer require --dev symfony/test-pack`.

Tests run in the `test` environment. Per-test config goes in
`config/packages/test/`; env vars go in `.env.test` (committed) or
`.env.test.local` (machine-specific). `.env.local` is intentionally ignored in
the test environment to keep setups consistent.

## Pick the right test type

- **Unit test** — one class/method in isolation, plain PHPUnit (`extends
  TestCase`). No kernel. Mirror the `src/` layout under `tests/` (a class in
  `src/Form/` is tested in `tests/Form/`).
- **Integration test** — a combination of classes, usually pulling a real
  service from the container. Extend `KernelTestCase`.
- **Application test** (a.k.a. functional test) — the whole app via simulated
  HTTP requests. Extend `WebTestCase`.

In large suites, group by subdirectory: `tests/Unit/`, `tests/Integration/`,
`tests/Application/`.

## Integration tests: KernelTestCase + the container

Boot the kernel, then fetch services from the special test container, which
exposes both public and non-removed private services:

```php
class NewsletterGeneratorTest extends KernelTestCase
{
    public function testSomething(): void
    {
        self::bootKernel();
        $container = static::getContainer();

        $generator = $container->get(NewsletterGenerator::class);
        $this->assertEquals('...', $generator->generateMonthlyNews()->getContent());
    }
}
```

The kernel is rebooted for each test, keeping them isolated. The kernel class
comes from the `KERNEL_CLASS` env var (set in `.env.test`).

**Mock a dependency** by setting it on the container before fetching the
service under test: `$container->set(NewsRepositoryInterface::class, $mock)`.
See [references/integration.md](references/integration.md).

## Application tests: WebTestCase + client

`createClient()` boots the kernel and returns a client that acts as a browser.
`request()` returns a `Crawler`.

```php
class PostControllerTest extends WebTestCase
{
    public function testIndex(): void
    {
        $client = static::createClient();
        $crawler = $client->request('GET', '/');

        $this->assertResponseIsSuccessful();
        $this->assertSelectorTextContains('h1', 'Hello World');
    }
}
```

- **Hardcode the request URL.** Do not generate it from the router — a raw URL
  makes the test fail when a route changes, signalling that you need a
  redirect for end users.
- Subsequent `request()` calls reboot the kernel (fresh container, cleared
  security token, detached Doctrine entities). Use one request per behavior, or
  see the docs on `disableReboot()` / `kernel.reset` if you need state to
  persist.
- Redirects are **not** followed automatically: use
  `$client->followRedirect()` after a redirect, or `followRedirects()` before
  the request to follow all.
- Log in without a login form via `$client->loginUser($testUser)` (load test
  users with fixtures). Does not work with stateless firewalls.
- `xmlHttpRequest()` for AJAX; pass server params to `createClient([], [...])`
  for custom HTTP headers (`HTTP_`-prefixed, uppercased).

### Asserting

Use any PHPUnit assertion plus Symfony's. Common ones (available on
`WebTestCase`):

- Response: `assertResponseIsSuccessful()`, `assertResponseStatusCodeSame(404)`,
  `assertResponseRedirects('/login')`, `assertResponseHeaderSame()`,
  `assertResponseHasCookie()`.
- Request/route: `assertRouteSame('app_post_show')`.
- Crawler/DOM: `assertSelectorExists()`, `assertSelectorCount()`,
  `assertSelectorTextContains('h1', '...')`, `assertSelectorTextSame()`,
  `assertPageTitleContains()`, `assertInputValueSame()`,
  `assertCheckboxChecked()`.

Full lists (mailer, notifier, http-client, browser assertions) in
[references/assertions.md](references/assertions.md).

## The DOM crawler

The crawler traverses the returned HTML/XML like jQuery; every method returns a
new `Crawler`, so chain them. Use it for richer assertions:

```php
$crawler = $client->request('GET', '/post/hello-world');
$this->assertCount(4, $crawler->filter('.comment'));
```

- Select: `filter('h1.title')` (CSS), `filterXpath('//h1')`, `eq(1)`,
  `first()`, `last()`, `children()`, `ancestors()`, `siblings()`.
- Extract: `text()`, `attr('href')`, `extract(['_text', 'href'])`,
  `each(fn ($node, $i) => ...)`. `count($crawler)` counts nodes.

**Click links / submit forms** — select a *button*, not a form (a form can have
several buttons):

```php
$client->clickLink('Click here');

$client->submitForm('Add comment', [
    'comment_form[content]' => '...',
]);
```

For finer control, get the `Link`/`Form` object via
`$crawler->selectLink('...')->link()` or `$crawler->selectButton('...')->form()`,
set field values (`->select()`, `->tick()`, `->upload()`), then
`$client->click($link)` / `$client->submit($form)`. See
[references/crawler.md](references/crawler.md).

## Database testing

- **Use a separate test database.** Set `DATABASE_URL` in `.env.test.local`
  (or `.env.test` if identical on every machine); a common convention is the
  `_test` suffix. Create it with
  `php bin/console --env=test doctrine:database:create` and
  `doctrine:schema:create`.
- **Keep tests independent.** Install `dama/doctrine-test-bundle` and enable it
  as a PHPUnit extension — it wraps each test in a transaction and rolls it
  back afterward.
- **Load fixtures** with `doctrine/doctrine-fixtures-bundle`
  (`make:fixtures`), then `php bin/console --env=test doctrine:fixtures:load`.
- **Test repositories against a real DB, not mocks.** Extend `KernelTestCase`,
  get the entity manager from the container, and close it in `tearDown()` to
  avoid memory leaks:

```php
protected function setUp(): void
{
    $kernel = self::bootKernel();
    $this->entityManager = $kernel->getContainer()->get('doctrine')->getManager();
}
```

Full repository example in [references/database.md](references/database.md).

## Best-practice rules

- **Smoke-test your URLs early.** A single data-provider-driven `WebTestCase`
  that requests every URL and asserts `assertResponseIsSuccessful()` catches
  broad breakage for little effort. Add it when you create the app, then layer
  specific tests on top. See [references/database.md](references/database.md).
- **Hardcode URLs in functional tests** (see above).
- Name test methods after the behavior they check (e.g.
  `testVisitingWhileLoggedIn`).
- Mocking Doctrine repositories in unit tests is **not** recommended;
  repositories belong in functional tests against a real connection.
