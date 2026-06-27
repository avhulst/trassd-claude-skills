---
name: shopware-app-scripts
description: >-
  Extend a Shopware App with app scripts — Twig-based server-side scripting for
  data loading, cart manipulation, and custom API endpoints, without PHP.
  Triggers when adding scripts under Resources/scripts in an app or hooking into
  cart/data/endpoint hooks (e.g. product-page-loaded, cart, api-*, store-api-*,
  storefront-*).
---

# Shopware App Scripts

App scripts let an app run logic **inside the Shopware execution stack** without
shipping PHP. They are [Twig](https://twig.symfony.com/) files executed in a
**sandboxed** environment, triggered by named **hooks**, with access to
pre-defined **services / facades** for reading data, manipulating the cart, and
producing API responses. Introduced in Shopware 6.4.8.0.

> Apps are sandboxed: scripts can only use the services exposed for the hook
> they run on, and the `repository` service only reads entities the app has been
> granted `read` permission for in `manifest.xml`.

## Directory convention — how scripts map to hooks

All scripts live under `Resources/scripts/` in the app. For each hook, create a
**subdirectory whose name matches the hook name**; drop one or more `.twig`
files inside. Every file in the directory runs when that hook fires (in
alphabetical order). Verified hook names from the framework:

```text
DemoApp/
├─ Resources/
│  └─ scripts/
│     ├─ product-page-loaded/      # data-loading hook → hook.page
│     │  └─ add-data.twig
│     ├─ cart/                      # cart manipulation hook → services.cart
│     │  └─ discount.twig
│     ├─ api-swag-export/           # Admin API endpoint  → /api/script/swag/export
│     ├─ store-api-swag-topseller/  # Store API endpoint  → /store-api/script/swag/topseller
│     ├─ storefront-swag-page/      # Storefront endpoint → /storefront/script/swag/page
│     ├─ response/                  # response-header hook (6.6.10.4+)
│     ├─ cache-invalidation/        # invalidate cached endpoint responses
│     └─ include/                   # reusable Twig macros (imported, not a hook)
└─ manifest.xml
```

There is no manifest registration step — placement in the correctly named
directory is what binds a script to a hook.

## Core rules

- **One subdirectory = one hook.** The directory name must equal the hook name
  exactly. Multiple `.twig` files in a directory all run for that hook.
- **Use `hook` for hook data, `services` for facades.** The `hook` variable
  carries the execution context (e.g. `hook.page`, `hook.context`,
  `hook.request`, `hook.query`). The `services` variable exposes the facades
  available for that hook (e.g. `services.repository`, `services.store`,
  `services.cart`, `services.config`, `services.response`, `services.cache`).
- **Not every service is available everywhere.** The set of `services.*`
  facades depends on the hook. `services.cart` exists only on the `cart` hook;
  `services.response` only on endpoint hooks.
- **Type hints aid the IDE.** Add `{# @var services \Shopware\Core\Framework\Script\ServiceStubs #}`
  at the top for autocompletion. The stub lists all services, but only the
  hook-relevant ones actually work.
- **`return` exits a script early; macros return values.** `{% return %}` stops
  the script; `{% return value %}` returns from a macro.
- **Prefix everything you name** (extensions, cart states, discount ids,
  endpoint hook names) with a vendor/app prefix to avoid collisions.

## Reusable scripts

Extract shared logic into [Twig macros](https://twig.symfony.com/doc/3.x/tags/macro.html)
under `Resources/scripts/include/`, then `{% import "include/x.twig" as x %}`.
The `include/` directory is not a hook, so its files never auto-execute.

## Interface hooks (blocks)

Some hooks are **interface hooks**: the script must implement named functions as
Twig `{% block %}`s. The `store-api-*` hook defines an optional `cache_key`
block and a **required** `response` block. Each block gets its own input data /
services, so what is available in `cache_key` differs from `response`. Omitting
a required block is an error.

```twig
{% block cache_key %}{# optional — build a unique key for the request #}{% endblock %}
{% block response %}{# required — produce the response #}{% endblock %}
```

## Data loading (Storefront pages)

On page-rendered hooks (e.g. `product-page-loaded`) the script receives the page
via `hook.page`, can load data, and attach it for the templates.

```twig
{# Resources/scripts/product-page-loaded/add-data.twig #}
{% set page = hook.page %}
{% set media = services.repository.search('media', { 'ids': [ page.product.customFields.myField ] }).first %}
{% do page.addExtension('swagMedia', media) %}
```

- `services.repository` → Admin-API-equivalent data; needs `read` permission per
  entity; methods `search()`, `ids()`, `aggregate()`.
- `services.store` → Store-API-equivalent data for *public* entities only
  (active/visible in the current sales channel); needs **no** extra permission;
  resolves SEO data and calculates product prices.
- Attach with `page.addExtension(name, struct)` (a `Struct`/entity/collection)
  or `page.addArrayExtension(name, {...})` (scalars or mixed JSON-like data).
  Read back in templates via `page.getExtension(name)`.

Criteria are built as JSON-like Twig objects (same shape as the API search
criteria): `ids`, `associations`, `filter`, `aggregations`, etc.

See [references/data-loading.md](references/data-loading.md).

## Cart manipulation

On the `cart` hook the script acts as an extra **cart processor** via the
`services.cart` fluent facade. It runs on **every cart calculation**, so make
actions idempotent (guard with `services.cart.has(id)` or a custom
`services.cart.states`).

```twig
{# Resources/scripts/cart/discount.twig #}
{% if not services.cart.has('swag-discount') %}
    {% do services.cart.discount('swag-discount', 'percentage', 10, 'my.label'|trans) %}
{% endif %}
```

Key operations: `products.add(id[, qty])`, `discount(id, 'absolute'|'percentage',
priceOrPercent, label)`, `remove(id)`, `price.create({...})`, `calculate()` (forced
recalculation — invalidates earlier references), `errors.error/notice/remove`,
`states`. Integrates with the Rule Builder via `hook.context.ruleIds`.

See [references/cart-manipulation.md](references/cart-manipulation.md).

## Custom API endpoints

A script directory prefixed with an API scope becomes a callable endpoint; the
rest of the directory name is the hook/route segment (`/` in the route maps to
`-` in the directory name):

| Directory prefix | Route                  | Methods     | Notes |
|------------------|------------------------|-------------|-------|
| `api-`           | `/api/script/{hook}`   | POST        | Admin API; no response caching |
| `store-api-`     | `/store-api/script/{hook}` | GET, POST | Interface hook; GET caching needs `cache_key` |
| `storefront-`    | `/storefront/script/{hook}` | GET, POST | Can render templates; GET cached by default |

Build the result with `services.response` and set it via `hook.setResponse(...)`;
default is `204 No Content`. Read input from `hook.request.*` (POST body) and
`hook.query` (GET params). `hook.stopPropagation()` halts remaining scripts.

```twig
{# Resources/scripts/store-api-swag-topseller/script.twig #}
{% block response %}
    {% set data = services.repository.aggregate('order', { ... }).first.jsonSerialize %}
    {% do hook.setResponse(services.response.json(data)) %}
{% endblock %}
```

`services.response` factory: `json(data)`, `redirect(route, params)`,
`render(template, data)` (storefront only). Tune caching on the response:
`response.cache.tag(...)`, `response.cache.disable()`,
`response.cache.sharedMaxAge(seconds)`. Invalidate via a `cache-invalidation`
script using `hook.event.getIds(entity)` and `services.cache.invalidate(tags)`.

The separate `response` hook (6.6.10.4+) manipulates HTTP headers on every
response: `hook.setHeader/getHeader`, `hook.routeName`, `hook.isInRouteScope(scope)`
(scopes: `storefront`, `store-api`, `api`, `administration`).

See [references/custom-endpoints.md](references/custom-endpoints.md).

## Debugging

In `APP_ENV=dev`, the Symfony debug toolbar lists triggered hooks and executed
scripts. Use `{% do debug.dump(value) %}` to dump data into the profiler.
