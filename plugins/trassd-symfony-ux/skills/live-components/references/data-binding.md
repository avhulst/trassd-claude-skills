# Data binding with `data-model`

`data-model` binds an HTML element to a writable `#[LiveProp]`. Live Components listens to the
field's event (usually `input`), updates the prop, and sends an AJAX request to re-render.

```twig
<input type="number" data-model="max">
```

The bound prop **must** be `#[LiveProp(writable: true)]` (read-only by default for security).

## Modifiers

Modifiers are piped before the model name (`modifier|model`):

- `debounce(N)` — wait N ms after the last keystroke before re-rendering. Re-rendering is already
  debounced ~150ms by default; this overrides it: `data-model="debounce(100)|max"`.
- `on(change)` — only re-render on the `change` event (after the field loses focus), not on every
  keystroke: `data-model="on(change)|max"`.
- `norender` — update the prop value in JavaScript but **don't** re-render yet. The new value is
  used on the next render (e.g. triggered by a button): `data-model="norender|coupon"`.

```twig
<input data-model="norender|coupon">
<button data-action="live#$render">Apply</button>   {# live#$render is the built-in re-render action #}
```

### Lightweight input validation modifiers

These gate the model update client-side (reducing requests):

- `min_length(N)` / `max_length(N)` — for text/email/password/search/url inputs and textareas.
- `min_value(N)` / `max_value(N)` — for number/range inputs.

```twig
<input type="search" data-model="min_length(3)|search">
<input type="number" data-model="min_value(10)|max_value(100)|quantity">
```

## Binding with `name=""` instead of `data-model`

Put `data-model` on the `<form>` and use `name` on each field — no per-field `data-model` needed.
`*` is the conventional value and modifiers still apply (`data-model="on(change)|*"`):

```twig
<form data-model="*">
    <input name="max" value="{{ max }}">
</form>
```

## Checkboxes, selects, radios, arrays

```php
#[LiveProp(writable: true)] public bool $agreeToTerms = false;
#[LiveProp(writable: true)] public array $foods = ['pizza', 'tacos'];
#[LiveProp(writable: true)] public string $meal = 'lunch';
```

```twig
{# checkbox without value => boolean; with value + name[] => array of strings #}
<input type="checkbox" data-model="agreeToTerms">
<input type="checkbox" data-model="foods[]" value="pizza">

{# radios set a single value; multi-select sets an array #}
<input type="radio" data-model="meal" value="lunch">
<select data-model="foods" multiple>...</select>
```

## Forcing a re-render explicitly

The built-in `live#$render` action re-renders on demand — pair it with `norender` inputs:

```twig
<input data-model="norender|coupon">
<button data-action="live#$render">Apply coupon</button>
```

## Updating a model without a form field

```twig
<button type="button" data-model="mode" data-value="edit" data-action="live#update">Edit</button>
```

## From custom JavaScript

Get the component object and set/render/act on it:

```javascript
import { getComponent } from '@symfony/ux-live-component';
this.component = await getComponent(this.element);
this.component.set('mode', 'editing');
this.component.render();
this.component.action('save', { arg1: 'value1' });
```

If an external library (e.g. a date picker) sets an input's value directly, the normal `change`
event may not fire, so the model won't update. Either set the model through the component object, or
dispatch the event yourself:

```javascript
input.value = 'sushi';
input.dispatchEvent(new Event('change', { bubbles: true }));
```

JavaScript lifecycle hooks are available via `this.component.on(...)`: `connect`, `disconnect`,
`render:started`, `render:finished`, `response:error`, `loading.state:started`,
`loading.state:finished`, `model:set`. (`render:*` fire only on re-render, not the initial render.)
