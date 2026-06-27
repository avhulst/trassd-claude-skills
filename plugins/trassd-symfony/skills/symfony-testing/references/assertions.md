# Symfony test assertions

Use any PHPUnit assertion plus the Symfony-provided ones below. They require the
listed base class / trait.

## Response assertions
(`WebTestCase`, `BrowserKitAssertionsTrait`, or `WebTestAssertionsTrait`)

- `assertResponseIsSuccessful()` — HTTP 2xx.
- `assertResponseStatusCodeSame(int $code)`.
- `assertResponseRedirects(?string $location, ?int $code)` — redirect (optionally
  check target and status); location may be absolute or relative.
- `assertResponseHasHeader($name)` / `assertResponseNotHasHeader($name)`.
- `assertResponseHeaderSame($name, $value)` / `...NotSame(...)`.
- `assertResponseHasCookie($name, $path, $domain)` / `...NotHasCookie(...)`.
- `assertResponseCookieValueSame($name, $value, $path, $domain)`.
- `assertResponseFormatSame(?string $format)`.
- `assertResponseIsUnprocessable()` — HTTP 422.

## Request assertions
(`WebTestCase`, `BrowserKitAssertionsTrait`, or `WebTestAssertionsTrait`)

- `assertRequestAttributeValueSame($name, $value)`.
- `assertRouteSame($expectedRoute, array $parameters = [])`.

## Browser assertions
(`WebTestCase`, `BrowserKitAssertionsTrait`, or `WebTestAssertionsTrait`)

- `assertBrowserHasCookie($name, ...)` / `assertBrowserNotHasCookie(...)`.
- `assertBrowserCookieValueSame($name, $value, ...)`.
- `assertBrowserHistoryIsOnFirstPage()` / `...IsNotOnFirstPage()`.
- `assertBrowserHistoryIsOnLastPage()` / `...IsNotOnLastPage()`.
- `assertThatForClient(Constraint $constraint)`.

## Crawler / DOM assertions
(`WebTestCase`, `DomCrawlerAssertionsTrait`, or `WebTestAssertionsTrait`)

- `assertSelectorExists($selector)` / `assertSelectorNotExists($selector)`.
- `assertSelectorCount(int $count, $selector)`.
- `assertSelectorTextContains($selector, $text)` / `...TextNotContains(...)`.
- `assertAnySelectorTextContains($selector, $text)` / `...NotContains(...)`.
- `assertSelectorTextSame($selector, $text)`.
- `assertAnySelectorTextSame($selector, $text)`.
- `assertPageTitleSame($title)` / `assertPageTitleContains($title)`.
- `assertInputValueSame($field, $value)` / `assertInputValueNotSame(...)`.
- `assertCheckboxChecked($field)` / `assertCheckboxNotChecked($field)`.
- `assertFormValue($formSelector, $field, $value)` / `assertNoFormValue(...)`.

## Mailer assertions
(`KernelTestCase` or `MailerAssertionsTrait`)

- `assertEmailCount(int $count, ?string $transport)`.
- `assertQueuedEmailCount(int $count, ?string $transport)`.
- `assertEmailIsQueued($event)` / `assertEmailIsNotQueued($event)`.
- `assertEmailAttachmentCount($email, int $count)`.
- `assertEmailTextBodyContains($email, $text)` / `...NotContains(...)`.
- `assertEmailHtmlBodyContains($email, $text)` / `...NotContains(...)`.
- `assertEmailHasHeader($email, $name)` / `assertEmailNotHasHeader(...)`.
- `assertEmailHeaderSame($email, $name, $value)` / `...NotSame(...)`.
- `assertEmailAddressContains($email, $name, $value)` / `...NotContains(...)`.
- `assertEmailSubjectContains($email, $value)` / `...NotContains(...)`.

Retrieve messages/events with `getMailerMessage($index, $transport)` and
`getMailerEvent($index, $transport)`.

## Notifier assertions
(`KernelTestCase` or `NotificationAssertionsTrait`)

- `assertNotificationCount(int $count, ?string $transportName)`.
- `assertQueuedNotificationCount(int $count, ?string $transportName)`.
- `assertNotificationIsQueued($event)` / `assertNotificationIsNotQueued($event)`.
- `assertNotificationSubjectContains($notification, $text)` / `...NotContains(...)`.
- `assertNotificationTransportIsEqual($notification, $name)` / `...IsNotEqual(...)`.

## HttpClient assertions
(`WebTestCase`, `HttpClientAssertionsTrait`, or `WebTestAssertionsTrait`)

Call `$client->enableProfiler()` before the code that triggers the HTTP
request(s).

- `assertHttpClientRequest($url, $method = 'GET', $body = null, $headers = [], $httpClientId = 'http_client')`.
- `assertNotHttpClientRequest($url, $method = 'GET', $httpClientId = 'http_client')`.
- `assertHttpClientRequestCount(int $count, $httpClientId = 'http_client')`.

## Profiler

`$client->enableProfiler()` before a request, then `$client->getProfile()`
afterward — useful to assert, e.g., a page runs under a query-count threshold.
