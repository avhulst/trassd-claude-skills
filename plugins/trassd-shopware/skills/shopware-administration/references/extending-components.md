# Overriding and extending components — detail

`Component.override()` overwrites the previous behavior of an existing component.
`Component.extend()` creates a **new** component based on an existing one. Both wrap
the `ComponentFactory` exposed on the global `Shopware` object.

## Override a template

```javascript
import template from './sw-text-field-new.html.twig';

Shopware.Component.override('sw-text-field', {
    template,
});
```

## Extend into a new component

```javascript
import template from './sw-custom-field.html.twig';

Shopware.Component.extend('sw-custom-field', 'sw-text-field', {
    template,
});
```

```twig
<sw-custom-field></sw-custom-field>
```

## TwigJS block customization

Given a core template that defines named blocks:

```twig
{% block card %}
    <div class="sw-card">
        {% block card_header %}
            <div class="sw-card--header">{{ header }}</div>
        {% endblock %}

        {% block card_content %}
            <div class="sw-card--content">{{ content }}</div>
        {% endblock %}
    </div>
{% endblock %}
```

Your override template redefines blocks. Redefining replaces the block; `{% parent %}`
re-renders the original markup:

```twig
{# replace this block entirely #}
{% block card_header %}
    <h1 class="custom-header">{{ header }}</h1>
{% endblock %}

{# keep original, then append #}
{% block card_content %}
    {% parent %}
    <div class="card-custom-content">...</div>
{% endblock %}
```

Override the **smallest** block that achieves your goal. Find the block name in the
core component's `.html.twig` (e.g. the dashboard headline lives in
`sw_dashboard_index_content_intro_content_headline`).

## Methods and computed properties — the `$super` call

Replacing a method entirely:

```javascript
Shopware.Component.extend('sw-custom-field', 'sw-text-field', {
    methods: {
        onInput() {
            // original onInput logic is fully replaced
        },
    },
});
```

Augmenting instead of replacing — call the original via `this.$super('<name>')`:

```javascript
Shopware.Component.extend('sw-custom-field', 'sw-text-field', {
    methods: {
        onInput() {
            const superCallResult = this.$super('onInput');
            // add custom logic
        },
    },
});
```

`$super` works for computed properties too:

```javascript
Shopware.Component.extend('sw-custom-field', 'sw-text-field', {
    computed: {
        stringRepresentation() {
            const superCallResult = this.$super('stringRepresentation');
            // add custom logic
        },
    },
});
```

## Wiring the override into the build

Place the registration in an `index.js`, then import it from `main.js`:

```javascript
// src/sw-dashboard-index-override/index.js
import template from './sw-dashboard-index.html.twig';

Shopware.Component.override('sw-dashboard-index', { template });
```

```javascript
// main.js
import './sw-dashboard-index-override/';
```

## Experimental: Composition API extension system

Shopware is introducing a Composition-API-based extension system for the future
migration away from the Options API. **The Options-API system (`override` / `extend`)
remains fully supported** — keep using it for Options-API components. Use the new
system only for components authored with the Composition API.

Two functions drive it:

- `Shopware.Component.createExtendableSetup` — used (mainly by core) to make a
  component extendable.
- `Shopware.Component.overrideComponentSetup` — used by plugins to override.

```javascript
Shopware.Component.overrideComponentSetup()('sw-product-list', (previousState, props, context) => {
    const newPageSize = ref(50);
    return {
        pageSize: newPageSize,   // override default page size
    };
});
```

The callback receives `previousState` (current reactive properties/methods), `props`,
and `context` (the Vue 3 setup context). Return an object of new or modified
properties/methods. You can mutate `previousState` (e.g. push a column) and return
`{}`.

Key differences from the Options-API system:

- Composition API syntax and Vue 3 reactive primitives instead of Vue 2 options.
- Function composition rather than option merging.
- **Only overrides are supported** — extending is done natively with the Composition
  API instead.
- Multiple plugins can override the same component; overrides apply in registration
  order. Maintain reactivity when changing reactive properties.

TypeScript is recommended; pass the component type as a generic
(`overrideComponentSetup<typeof Component>()`) for typed `previousState` and `props`.
Private properties are reachable via `previousState._private` but this is untyped and
not recommended for production.
