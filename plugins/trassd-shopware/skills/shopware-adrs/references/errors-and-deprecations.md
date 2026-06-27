# Errors, deprecations & feature lifecycle

## Domain exceptions (2022-02-24)

**Decision:** Each domain gets a single exception class extending
`Shopware\Core\Framework\HttpException`, with a `private` constructor and a
static factory method per error case. Every case has a unique string error code
(e.g. `CMS_NOT_FOUND`) that is surfaced to API consumers. Bare
`\RuntimeException` / `\InvalidArgumentException` are discouraged because they
are untraceable. Exceptions you intend to `catch` get their own subclass
(extending the domain exception) in an `Exception/` subfolder.

**Rule for extensions:** Define one `MyPluginException` class with named
factory methods and unique codes; throw `MyPluginException::somethingFailed()`.
Only create a dedicated subclass when you genuinely catch that specific case.

```php
class CmsException extends HttpException
{
    public const NOT_FOUND_CODE = 'CMS_NOT_FOUND';

    public static function notFound(?\Throwable $e = null): self
    {
        return new self(Response::HTTP_NOT_FOUND, self::NOT_FOUND_CODE, 'Cms page not found', [], $e);
    }
}
```

## Consistent deprecation notices (2022-02-28)

**Decision:** Every `@deprecated` annotation is paired with a runtime
deprecation notice via `Feature::triggerDeprecationOrThrow()` (which throws when
the next-major flag is active, so core never relies on its own deprecated code).
Notices may be triggered conditionally (only when the method is called the old
way). Messages must name the method/class, the removal version, and the
replacement.

**Rule for extensions:** When you deprecate your own API, add both the
`@deprecated tag:vX` annotation and a runtime deprecation. Write actionable
messages: "Method `OldFeature::method()` will be removed in v6.5.0.0, use
`NewFeature::method()` instead", not "Will be removed". Run your plugin tests
against a new Shopware version to surface the deprecations you depend on.

## Exception log levels (2023-05-25)

**Decision:** By default uncaught exceptions log at `error`. Expected client
errors (4xx `ShopwareHttpException`) are noise, so the platform degrades their
log level via Symfony's `framework.exceptions` configuration — keyed by the
Shopware **error code** (since one domain-exception class holds many cases),
not by FQCN.

**Rule for extensions:** For your own expected-client-error codes, lower the log
level through the `exceptions` configuration so genuine errors stay visible.
Projects can override the level per code when they need to debug clients.

## Experimental features (2023-05-10)

**Decision:** Code that isn't yet API-stable is marked
`@experimental stableVersion:vX.Y.Z` and is explicitly **not** covered by the BC
promise — the implementation can change freely until that version, at which
point the annotation must be removed (or the feature deprecated/killed). Code
that must never become public uses `@internal` instead. Experimental features
are still production-ready; data is migrated, not discarded. The same annotation
works for PHP, JS (`@experimental stableVersion:v6.6.0`) and Twig blocks
(`{# @experimental stableVersion:v6.6.0 #}`).

**Rule for extensions:** Do not build against a core symbol marked
`@experimental` as if it were stable — it may change in any minor release.
Conversely, when you ship an early increment of your own feature, mark it
`@experimental` with a target stable version so consumers know the API isn't
frozen.

## Feature flags for major versions (2022-01-20)

**Decision:** Breaking changes for the next major are merged early but hidden
behind a single major feature flag (`v6.5.0.0`, `v6.6.0.0`, …), toggled via env
(`V6_5_0_0=1`) and checked with the `Feature` class. The flag persists after
release, so it doubles as a version switch.

**Rule for extensions:** Hide your own next-major-only code behind the major
flag with `Feature::isActive('v6.6.0.0')` / `Feature::ifActive(...)`, flag admin
modules (`flag: 'v6.6.0.0'`) and Twig (`{% if feature('v6.6.0.0') %}`), and skip
flagged tests with `Feature::skipTestIfActive(...)`. You can read the major flag
to support several Shopware majors from one plugin version.

```php
if (Feature::isActive('v6.6.0.0')) {
    // next-major behaviour
}
```

## PHP enums (2023-05-16)

**Decision:** New collections of constant values use PHP enums instead of class
constants plus validity-checking arrays. Existing constant lists migrate via the
expand-and-contract pattern: accept `Enum|string`, cast strings to the enum,
deprecate the constants and the string path, then narrow the signature to the
enum only.

**Rule for extensions:** Model fixed value sets (types, statuses) as enums.
When evolving a public API toward an enum, accept `MyEnum|string` first and
deprecate the string before requiring the enum, so you don't break callers.

```php
enum IndexMethod { case PARTIAL; case FULL; }

public function product(int $id, IndexMethod $method): void
{
    match ($method) {
        IndexMethod::PARTIAL => $this->partial($id),
        IndexMethod::FULL    => $this->full($id),
    };
}
```

## Make Rule classes internal (2025-01-29)

**Decision:** Core Rule classes are being marked `@internal` so the rule system
can evolve without breaking third-party extensions. A small set of
config-driven rules stays public (e.g. `LineItemOfTypeRule`,
`LineItemProductStatesRule`, `PromotionCodeOfTypeRule`, `ZipCodeRule`,
`BillingZipCodeRule`, `ShippingZipCodeRule`) because they are reasonably
expected to be extended via configuration.

**Rule for extensions:** Don't extend or directly use internal core rule
classes — create a new rule class for custom conditions. Treat internal rule
implementations as subject to change.
</content>
