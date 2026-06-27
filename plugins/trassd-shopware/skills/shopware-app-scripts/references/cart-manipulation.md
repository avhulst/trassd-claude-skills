# Cart Manipulation with App Scripts

Scripts in `Resources/scripts/cart/` run on the `cart` hook and act as an extra
**cart processor**. They execute on **every cart calculation** (adding items,
changing shipping/payment, etc.), so they must be idempotent. Use the
`services.cart` fluent facade.

## Idempotency

Guard your changes so repeated runs don't stack:

```twig
{# guard by checking the line item exists #}
{% if not services.cart.has('swag-discount') %}
    {% do services.cart.discount('swag-discount', 'percentage', 10, 'A 10% discount') %}
{% endif %}
```

Or track your own state so you can also revert it:

```twig
{% set isEligible = services.cart.items.count > 3 %}

{% if not services.cart.states.has('swag-my-state') %}
    {% if isEligible %}{# perform action #}{% endif %}
{% else %}
    {% if not isEligible %}{# revert action #}{% endif %}
{% endif %}
```

State names must be unique — use a vendor prefix.

## Recalculating

Price totals are recalculated automatically after the whole script finishes. If
later steps in the same script depend on updated totals, force it:

```twig
{% do services.cart.products.add(productId) %}
{% do services.cart.calculate() %}
```

> `calculate()` re-runs the full process step and **recreates** the cart's
> properties (`products()`, `items()`, `price()`). Any variables holding
> references from before the call are stale afterward.

## Price definitions

Prices are gross/net and currency-dependent. Create one with the factory:

```twig
{% set price = services.cart.price.create({
    'default': { 'gross': 19.99, 'net': 19.99 },
    'EUR':     { 'gross': 19.99, 'net': 19.99 },
    'USD':     { 'gross': 24.99, 'net': 21.37 }
}) %}
```

Prices can also come from `price`-type custom fields or app config:

```twig
{% set priceData = services.config.app('myCustomPrice') %}
{% set discountPrice = services.cart.price.create(priceData) %}
```

Add a null-check when the config has no default value.

## Line items

```twig
{# add a product (optional quantity) #}
{% do services.cart.products.add(productId) %}
{% do services.cart.products.add(productId, 4) %}

{# absolute discount — id, type, price definition, label #}
{% if not services.cart.has('swag-abs') %}
    {% do services.cart.discount('swag-abs', 'absolute', discountPrice, 'my.label'|trans) %}
{% endif %}

{# relative discount — percentage instead of a price #}
{% do services.cart.discount('swag-rel', 'percentage', 10, 'A custom 10% discount') %}

{# remove by id (product id or discount id) #}
{% do services.cart.remove(productId) %}
{% do services.cart.remove('swag-rel') %}
```

Split a line item with `take(qty[, newId])` — it returns the split item, which
you must add back yourself:

```twig
{% set existing = services.cart.products.get(productId) %}
{% if existing and existing.quantity > 3 %}
    {% set newItem = existing.take(2, newLineItemId) %}
    {% do services.cart.products.add(newItem) %}
{% endif %}
```

Custom payload on a line item:

```twig
{% set lineItem = services.cart.get(lineItemId) %}
{% do lineItem.payload.set('custom-payload', myValue) %}
{% set value = lineItem.payload['custom-payload'] %}
```

## Errors and notifications

Errors block checkout; warnings/notices only inform. First arg is the snippet
key; optional second arg is an id (to remove later); optional third is snippet
params.

```twig
{% if not cartIsValid %}
    {% do services.cart.errors.error('my-error-message', 'error-id') %}
{% else %}
    {% do services.cart.errors.remove('error-id') %}
{% endif %}

{% do services.cart.errors.notice('my-notice') %}
```

## Rule-based scripts

Cart scripts integrate with the Rule Builder. Store a chosen rule id in app
config and check it against the matched rules on the context:

```twig
{% set ruleId = services.config.app('exampleRule') %}

{% if ruleId and ruleId in hook.context.ruleIds %}
    {# perform action #}
{% else %}
    {# revert action #}
{% endif %}
```
