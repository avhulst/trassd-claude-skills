# Loading states, deferred/lazy loading, polling, URL binding

## Loading states

Show/hide or restyle elements while the component re-renders or an action runs.

```twig
<span data-loading>Loading</span>              {# show while loading (default) #}
<span data-loading="hide">Saved!</span>        {# hide while loading #}
<div data-loading="addClass(opacity-50)">...</div>
<div data-loading="removeClass(opacity-50)">...</div>
<div data-loading="addAttribute(disabled)">...</div>   {# only empty attrs: disabled/readonly/required #}
```

Combine directives with spaces, and add `delay` (default 200ms) to avoid flicker on fast requests:

```twig
<div data-loading="delay|addClass(opacity-50)">...</div>
<div data-loading="delay(500)|show">Loading</div>
```

Scope the loading behavior:

- To a specific action: `data-loading="action(saveForm)|show"`.
- To a specific model change: `data-loading="model(email)|show"` (supports child paths like
  `model(user.email)`).

### `aria-busy`

While re-rendering/processing, the root element automatically gets `aria-busy="true"` (removed when
done). This aids screen readers and lets you style the loading state in CSS without `data-loading`,
e.g. Tailwind's `aria-busy:opacity-50` or `[aria-busy="true"] { opacity: .5 }`.

## Deferred and lazy loading

Both render an empty `<div>` first, then fetch the real component via AJAX. The component **is**
created and mounted on initial load — only its template render is deferred — so keep heavy work in
methods (e.g. `getProducts()`) that are only called from the template.

- `loading="defer"` — render right after the page loads. Use for heavy components needed on load.
- `loading="lazy"` — render when the element scrolls into the viewport. Use for components far down
  the page that may never be seen.

```twig
<twig:SomeHeavyComponent loading="defer" />
{{ component('SomeHeavyComponent', { loading: 'lazy' }) }}
```

### Loading placeholders

Define a `placeholder(props)` macro **outside** the main content in the component template; it
renders while deferred and receives the passed props:

```twig
<div {{ attributes }}>
    {% for product in this.products %}<div>{{ product.name }}</div>{% endfor %}
</div>

{% macro placeholder(props) %}
    {% for i in 1..props.size %}<div class="loading-product">...</div>{% endfor %}
{% endmacro %}
```

From the calling template you can instead use `loading-template="spinner.html.twig"` or override the
`loadingContent` block (these take precedence over the `placeholder` macro). Change the placeholder
wrapper tag with `loading-tag="span"`.

## Polling

Add `data-poll` to the **root** element to re-render every 2s. Use `delay(N)` to change the
interval (then you must name `$render` explicitly), or name an action to call instead:

```twig
<div {{ attributes }} data-poll>...</div>
<div {{ attributes }} data-poll="delay(500)|$render">...</div>
<div {{ attributes }} data-poll="delay(2000)|save">...</div>
```

## Binding a LiveProp to the URL

`url: true` syncs a writable prop with a query parameter (via `history.replaceState()`, so no new
history entry). Loading a URL with that parameter initializes the prop (the value passes through
hydration; un-hydratable values are ignored).

```php
#[LiveProp(writable: true, url: true)]
public string $query = '';
```

- Scalars, arrays, and objects are all supported (`prop[0]=...`, `prop[foo]=...`).
- With multiple URL-bound props, **all** of them are written to the URL on any change, even unchanged
  ones.
- Rename the parameter with `UrlMapping`: `#[LiveProp(writable: true, url: new UrlMapping(as: 'q'))]`.
- Map to a **route path** segment instead of a query param with `new UrlMapping(mapPath: true)` (the
  route must define that `{param}`; otherwise it falls back to a query param).
- Per-page parameter names: combine `url: true` with a `modifier` method that returns
  `$liveProp->withUrl(new UrlMapping(as: ...))` — lets you use the same component multiple times
  without name collisions.

### Validate URL-bound props

The initial value from the URL is **not** auto-validated. Validate it in a `#[PostMount]` hook using
`ValidatableComponentTrait`:

```php
#[PostMount]
public function postMount(): void
{
    if (!$this->validateField('mode', false)) { /* handle invalid */ }
}
```

## Customizing the component route & fetch credentials

- `#[AsLiveComponent(route: 'live_component_admin')]` uses a custom route (declare it with
  `_live_component`/`_live_action` placeholders) — e.g. to place the component behind a firewall.
  `urlReferenceType:` controls absolute vs relative URL generation.
- `#[AsLiveComponent(fetchCredentials: 'include')]` — `same-origin` (default), `include`, or `omit`,
  mirroring the fetch() `credentials` option, for cross-origin requests that need credentials.
