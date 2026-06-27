# Apps

## App scripts (2021-10-21)

**Decision:** Apps run synchronous logic via Twig-based **app scripts** placed
in `Resources/scripts/<hook-folder>/`, subscribed to named scripting hook points
(rules, cart, storefront page loading, shipping calculation, flow builder, …).
Twig provides a secure sandbox: scripts have no direct DB or filesystem access.
Core objects passed to scripts are either plain `Struct` data containers
(injected directly) or services/business-logic structs (like `Cart`) wrapped in
a tailored **facade** that limits and stabilizes the API. Global helpers such as
`dal_search` are available; each script runs in its own reduced Twig environment.

**Rule for extensions:** Implement synchronous app behaviour as scripts in the
correctly named hook folder and interact only through the provided facades and
globals (e.g. `cart.discount(...)`, `cart.block(...)`). Treat the facade methods
as a long-lived, BC-governed API; the Shopware version is injected into the
script context so you can guard newer features.

```twig
{% if cart.price.totalPrice < 500 %}
    {% do cart.block('you have to pay at least 500€ for this cart') %}
{% endif %}
```

## Custom app API endpoints (2022-01-06)

**Decision:** Apps expose custom endpoints without manifest configuration via
three generic routes — `/api/script/{hook}`, `/store-api/script/{hook}` and
`/storefront/script/{hook}` — where `{hook}` names the script hook (prefixed
`api-`, `store-api-`, `storefront-`). Scripts receive the request data and
context (storefront additionally gets `query` and a `GenericPage`). A `response`
service builds the reply (`json`, `redirect`, `render`), assigned via
`hook.setResponse(...)`; no response yields HTTP 204. Storefront/store-api
routes are cached by default (opt-out via the response cache API);
`/api` routes are never cached. `hook.context.ensureLogin()` enforces a
logged-in customer. Multiple scripts can run per hook unless one calls
`hook.stopPropagation()`.

**Rule for extensions:** Build custom endpoints entirely inside app scripts (the
`Resources/scripts` folder), not in `manifest.xml`. Always vendor-prefix your
hook name to avoid other apps overriding it. Use `services.response.*` to return
JSON, redirect, or render a Twig template, and `response.cache.*` to tune
caching.

```twig
{% set response = services.response.json({'data': data}, statusCode) %}
{% do hook.setResponse(response) %}
```
</content>
