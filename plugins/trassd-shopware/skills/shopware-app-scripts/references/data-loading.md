# Data Loading with App Scripts

Page-rendered hooks let you enrich the data passed to Storefront templates. The
hook gives you the `page` object via `hook.page`; you load data with the
`repository` / `store` services and attach it as an **extension** the template
can read back.

## Attach data to the page

```twig
{# Resources/scripts/product-page-loaded/my-script.twig #}
{# @var services \Shopware\Core\Framework\Script\ServiceStubs #}
{% set page = hook.page %}

{# the page exposes all page data, e.g. the current product #}
{% set extra = { 'example': 'just an example' } %}

{% do page.addArrayExtension('swagMyData', extra) %}
```

Read it back in a template:

```twig
{# Resources/views/storefront/page/product-detail/index.html.twig #}
{% sw_extends '@Storefront/storefront/page/product-detail/index.html.twig' %}

{% block page_product_detail %}
    <h1>{{ page.getExtension('swagMyData').example }}</h1>
    {{ parent() }}
{% endblock %}
```

## `repository` vs `store`

| Aspect | `services.repository` | `services.store` |
|--------|----------------------|------------------|
| Data equivalent | Admin API (CRUD) | Store API |
| Entities | any entity | only "public" entities (active + visible in current sales channel) |
| Permissions | needs `read` per entity in `manifest.xml` | no extra permission |
| Extras | raw data | resolves SEO data, calculates product prices |

Both expose `search()`, `ids()`, `aggregate()`. Pass the entity name first, then
a criteria object:

```twig
{% set criteria = {
    'ids': [ 'id1', 'id2' ],
    'associations': { 'manufacturer': {}, 'cover': {} },
    'filter': [ { 'type': 'equals', 'field': 'active', 'value': true } ]
} %}

{% set products = services.repository.search('product', criteria) %}
```

The criteria object mirrors the JSON criteria used by the API (filters,
associations, aggregations, sorting, pagination).

## addExtension vs addArrayExtension

- `page.addExtension(name, struct)` — `struct` must be a PHP `Struct` (an entity
  or collection returned by the services). Use for a single object/collection.
- `page.addArrayExtension(name, {...})` — wrap scalars or multiple values in a
  JSON-like Twig object.

```twig
{% set products = services.repository.search('product', criteria) %}

{% do page.addExtension('swagCollection', products) %}
{% do page.addExtension('swagEntity', products.first) %}

{% do page.addArrayExtension('swagArray', {
    'collection': products,
    'entity': products.first,
    'scalar': 'a value'
}) %}
```

```twig
{# in the template #}
{% for product in page.getExtension('swagCollection') %}{% endfor %}
{% set product = page.getExtension('swagArray').entity %}
<h1>{{ page.getExtension('swagArray').scalar }}</h1>
```

Extension names must be unique — always use a vendor prefix. Extensions can be
added to any `Struct`, not only the page (e.g. to each product on the page).
