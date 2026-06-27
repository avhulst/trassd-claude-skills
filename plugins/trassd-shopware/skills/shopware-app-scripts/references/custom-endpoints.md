# Custom API Endpoints with App Scripts

Prefix a script directory with an API scope to expose it as an HTTP endpoint.
The remainder of the directory name is the route segment; slashes in the route
map to dashes in the directory name (so `swag/topseller` ⇒ `swag-topseller`).

| Directory prefix | Route                       | Methods   | Caching |
|------------------|-----------------------------|-----------|---------|
| `api-`           | `/api/script/{hook}`        | POST      | not supported |
| `store-api-`     | `/store-api/script/{hook}`  | GET, POST | GET cacheable, requires `cache_key` |
| `storefront-`    | `/storefront/script/{hook}` | GET, POST | GET cached by default (HTTP cache) |

`Resources/scripts/store-api-swag-topseller/` is reachable at both
`/store-api/script/swag/topseller` and `/store-api/script/swag-topseller`.
Always include a vendor/app prefix in the hook name to avoid collisions.

## Input and response

- `hook.request.*` — POST body parameters.
- `hook.query` — GET query parameters.
- `hook.setResponse(response)` — set the response (default is `204 No Content`).
- `hook.stopPropagation()` — skip remaining scripts in the directory (they run
  in alphabetical order; later scripts can otherwise override the response).

The `services.response` factory builds responses:

- `services.response.json(data)`
- `services.response.redirect(routeName, params)`
- `services.response.render(template, data)` — Storefront scope only.

## Admin API (`api-`)

POST only, no response caching.

```twig
{# Resources/scripts/api-swag-export/script.twig #}
{% set response = services.response.json({ 'foo': 'bar' }) %}
{% do hook.setResponse(response) %}
```

## Store API (`store-api-`) — interface hook

This is an interface hook: implement a required `response` block and an optional
`cache_key` block.

```twig
{# Resources/scripts/store-api-swag-topseller/script.twig #}
{% block response %}
    {% set categoryId = hook.request.categoryId %}
    {% set criteria = {
        aggregations: [ {
            name: "categoryFilter", type: "filter",
            filter: [ { type: "equals", field: "order.lineItems.product.categoryIds", value: categoryId } ],
            aggregation: {
                name: "orderedProducts", type: "terms", field: "order.lineItems.productId",
                aggregation: { name: "quantityItemsOrdered", type: "sum", field: "order.lineItems.quantity" }
            }
        } ]
    } %}
    {% set agg = services.repository.aggregate('order', criteria) %}
    {% do hook.setResponse(services.response.json(agg.first.jsonSerialize)) %}
{% endblock %}

{% block cache_key %}
    {% set payload = hook.query|merge({ 'script': 'topseller' }) %}
    {% do hook.setCacheKey(payload|md5) %}
{% endblock %}
```

The app needs `read` permissions for every entity it queries (e.g. `order`,
`order_line_item`, `product`) declared in `manifest.xml`.

## Storefront (`storefront-`)

Can render Twig templates or redirect, in addition to JSON. Note that a
Storefront endpoint makes the app incompatible with headless consumers.

```twig
{# render a template #}
{% set product = services.store.search('product', { 'ids': [productId] }).first %}
{% do hook.page.addExtension('myProduct', product) %}
{% do hook.setResponse(
    services.response.render('@MyApp/storefront/page/custom-page/index.html.twig', { 'page': hook.page })
) %}
```

```twig
{# redirect to an existing route #}
{% set response = services.response.redirect('frontend.detail.page', { 'productId': hook.query['product-id'] }) %}
{% do hook.setResponse(response) %}
```

## Response caching

Configure caching on the response object (customer-facing APIs only):

```twig
{% set response = services.response.json({ 'foo': 'bar' }) %}
{% do response.cache.tag('my-custom-tag') %}   {# tag for fine-grained invalidation #}
{% do response.cache.disable() %}              {# opt out of caching #}
{% do response.cache.sharedMaxAge(120) %}      {# s-maxage in seconds (6.7.6.0+) #}
{% do hook.setResponse(response) %}
```

> `cache.maxAge()` is deprecated (removed in 6.8.0.0); use `sharedMaxAge()`.

## Cache invalidation

A `cache-invalidation` script inspects write operations and invalidates tags.

```twig
{# Resources/scripts/cache-invalidation/invalidate.twig #}
{% set ids = hook.event.getIds('product') %}
{% set ids = ids.only('insert').with('description', 'parentId') %}
{% if ids.empty %}{% return %}{% endif %}

{% set tags = [] %}
{% for id in ids %}{% set tags = tags|merge(['my-product-' ~ id]) %}{% endfor %}
{% do services.cache.invalidate(tags) %}
```

`getIds(entity)` returns the written ids; filter with `only('insert'|'update'|
'delete')` and `with(...changedProperties)` (chainable).

## Response-header hook (6.6.10.4+)

A dedicated `response` hook adjusts HTTP headers on every response:

```twig
{# Resources/scripts/response/response.twig #}
{% if hook.routeName == 'frontend.detail.page' and hook.isInRouteScope('store-api') %}
    {% do hook.setHeader('X-Frame-Options', 'SAMEORIGIN') %}
{% endif %}
```

Route scopes: `storefront`, `store-api`, `api`, `administration`. Read current
values with `hook.getHeader(name)`.
