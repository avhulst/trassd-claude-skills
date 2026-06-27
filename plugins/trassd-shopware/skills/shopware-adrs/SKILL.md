---
name: shopware-adrs
description: >-
  Apply Shopware's binding Architecture Decision Records (ADRs) when designing
  or extending Shopware — the decoration pattern, SQL vs DAL, creating events,
  domain exceptions & deprecation handling, feature flags/experimental code, PHP
  enums, nested line items, payment/tax/checkout gateways, app scripting, and
  storefront/admin standards. Triggers when making an architectural decision in
  a Shopware extension or checking a design against the platform's documented
  conventions.
---

# Shopware ADRs

Architecture Decision Records (ADRs) capture *why* Shopware's core was built a
certain way and *how* it must be extended. They are not suggestions: every
non-`@internal` API ships under a backwards-compatibility promise, and the ADRs
define the patterns that promise is built on. When you design a service,
exception, event, payment flow, or storefront/admin extension, your design must
match the relevant ADR or you will fight the platform (broken decoration
chains, untraceable exceptions, BC breaks for your own consumers).

An ADR documents requirements, affected domains, the decision, *and* the
consequences for developers who use the code (see
`resources/guidelines/code/core/adr.md`). The rules below distill the binding
developer-facing decisions. Each links to a grouped reference with the
decision, the rule for extension developers, and a short example.

## Core architecture & the DAL

See [references/core-and-dal.md](references/core-and-dal.md).

- **Decorate against abstract classes, never interfaces, and always implement
  `getDecorated()`** — abstract classes let signatures evolve without breaking
  the decoration chain. (2020-11-25)
- **Use the DAL for any read/write that third parties may extend** (Store-API,
  storefront page loaders, admin API, all writes); reach for plain SQL only in
  entity indexers and internal core components that are not extension points.
  (2021-05-14)
- **To filter a to-many association on several values that relate to each
  other, wrap the filters in one multi-filter** — separate `addFilter` calls
  create separate joins and change the result set. (2020-11-19)
- **Define app entities in `config/custom_entity.xml` with a vendor prefix**
  (`custom_entity_` / `ce_`); fields must be nullable or defaulted, you get an
  `id` + translated `label`, and only `store_api_aware` entities reach the
  Store-API. (2021-09-14)
- **Never rely on `autoload: true` associations; request what you need via the
  `Criteria`** — autoloaded associations are deprecated for performance.
  (2023-02-02)
- **Generate IDs with the `Uuid` class (UUIDv7)** — keep using it; do not roll
  your own random UUIDs. (2023-05-22)
- **Prefer the event-based extension system over new
  decoration/adapter/factory layers** where an `Extension` point exists; hook
  the `.pre`/`.post` events and `stopPropagation()` to replace core behaviour.
  (2024-06-18)

## Errors, deprecations & feature lifecycle

See [references/errors-and-deprecations.md](references/errors-and-deprecations.md).

- **Throw a domain exception via a static factory on one per-domain class
  extending `HttpException`**, each with a unique error code — never throw a
  bare `\RuntimeException`. (2022-02-24)
- **Pair every `@deprecated` annotation with a runtime
  `Feature::triggerDeprecationOrThrow()`**, and write the message as
  "`Class::method()` will be removed in vX, use Y instead". (2022-02-28)
- **Let expected client errors (4xx) log below `error` level** via Symfony's
  `framework.exceptions` config keyed by the Shopware error code, so real errors
  stay visible. (2023-05-25)
- **Mark unstable APIs `@experimental stableVersion:vX.Y.Z`** (not BC-covered),
  and `@internal` for code that must never become public API. (2023-05-10)
- **Gate next-major changes behind the major feature flag** (e.g. `v6.6.0.0`)
  using the `Feature` helper; plugins can read the same flag to support multiple
  majors. (2022-01-20)
- **Represent a fixed set of constants with a PHP enum, not class constants**;
  migrate via the expand-and-contract pattern (`Enum|string` then enum-only).
  (2023-05-16)
- **Treat Rule classes as `@internal` — create a new rule class, never extend a
  core one** (a few config-driven rules stay public). (2025-01-29)

## Extension patterns

See [references/extension-patterns.md](references/extension-patterns.md).

- **Configure controller behaviour through `#[Route]` `defaults`
  (`_loginRequired`, `_acl`, `_routeScope`, …), not custom class annotations** —
  defaults survive decoration. (2022-02-09)
- **Make repository-dependent classes testable with `StaticEntityRepository`**
  instead of hand-built `EntitySearchResult` mocks. (2023-04-01)
- (Decoration vs events: see Core and Errors sections above — choose the
  event-based extension point when one exists.)

## Checkout

See [references/checkout.md](references/checkout.md).

- **Implement payments as a synchronous or asynchronous payment handler**, and
  add pre-created-payment support for headless flows; failed payments enter the
  after-order retry loop. (2021-10-01)
- **Override taxes through a `shopware.tax.provider`-tagged service implementing
  `TaxProviderInterface`**, selected by `priority` + availability rule, run after
  cart calculation. (2022-04-28)
- **Influence available payment/shipping methods and block carts via a
  `CheckoutGatewayInterface` implementation** (or an app `gateways/checkout`
  endpoint returning gateway commands). (2024-04-01)
- **Manipulate stock through `AbstractStockStorage` (`load`/`alter`/`index`),
  not by writing `available_stock`** — `stock` is now the realtime value; custom
  inventory replaces or decorates the storage. (2023-05-15)

## Apps

See [references/apps.md](references/apps.md).

- **Add synchronous app logic as Twig app scripts in `Resources/scripts/`
  subscribed to named hook points** — sandboxed, no direct DB/filesystem access.
  (2021-10-21)
- **Expose custom app endpoints via `/api|store-api|storefront/script/{hook}`
  with a vendor-prefixed hook name**, building the response with the `response`
  service — no extra `manifest.xml` config needed. (2022-01-06)

## Storefront & admin

See [references/storefront-and-admin.md](references/storefront-and-admin.md).

- **Render every line item type through the recursive base template
  `component/line-item/line-item.html.twig`** instead of the old per-area
  checkout-item templates. (2022-03-17)
- **Keep storefront controllers thin**: `frontend.*` route names, a PageLoader
  for page data, a Store-API route behind every action, no business logic and no
  direct repository use in the controller. (2021-08-10)
- **Write new storefront JS as TypeScript** (`.ts`/`.tsx`, Babel chain); never
  rename a public `.js` to `.ts` without deprecation. (2022-06-24)
- **Extend the admin via component sections and UI extension points
  (`sw.ui.*`, `sw.location`, `sw.ui.componentSection`)** identified by
  position/location IDs — not by patching core components. (2021-12-07)
- **Build admin state with Pinia under `Shopware.Store`**, not Vuex /
  `Shopware.State`; stores are TypeScript and export a state type. (2024-06-17)

## Testing

See [references/testing.md](references/testing.md).

- **Follow the test pyramid**: many unit tests, few high-value E2E tests; cover
  CRUD/logic with unit or PHP integration tests rather than E2E. (2023-02-13)
- (Mocking repositories with `StaticEntityRepository` — see Extension patterns.)
</content>
</invoke>
