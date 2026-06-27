# HTML syntax, anonymous & wrapper components

## HTML syntax recap

```html+twig
<twig:Alert message="Hi" withCloseButton />     {# bare attr = boolean true #}
<twig:Alert :user="user.id" />                   {# ":" prefix = dynamic expression #}
<twig:Alert user="{{ user.id }}" />              {# equivalent to the above #}
<twig:Alert :foo="{col: ['a','b']}" />           {# pass arrays/objects #}
<twig:Alert {{ ...myAttributes }} />             {# spread an array (Twig >= 3.7) #}
```

Boolean gotcha: the string `"false"` juggles to boolean `true`. To pass real
`false`, use `:withCloseButton="false"` or `withCloseButton="{{ false }}"`.

## Anonymous components

A component with no PHP logic needs only a template under
`templates/components/`. Its **name is its location**:

- `templates/components/Button/Primary.html.twig` → `<twig:Button:Primary>`.
- A directory can use `index.html.twig` to avoid repetition:
  `templates/components/Menu/index.html.twig` → `<twig:Menu>`, and
  `templates/components/Menu/Item.html.twig` → `<twig:Menu:Item>`.

Declare props with the `{% props %}` tag at the top. Props are **required by
default**; give a default with `=`. Anything not declared as a prop is treated
as an attribute.

```html+twig
{# templates/components/Button.html.twig #}
{% props icon, type = 'primary' %}   {# icon required, type defaults #}
<button {{ attributes.defaults({class: 'btn btn-' ~ type}) }}>
    {% block content %}{% endblock %}
    {% if icon %}<span class="fa-{{ icon }}"></span>{% endif %}
</button>
```

```html+twig
<twig:Button>Share</twig:Button>                 {# error: 'icon' missing #}
<twig:Button icon="share">Share</twig:Button>    {# type uses its default #}
<twig:Button icon="share" type="secondary">Share</twig:Button>
```

Extra attributes still render on the element:
`<twig:Button:Primary type="button" name="foo">` →
`<button class="primary" type="button" name="foo">`.

## Third-party bundle components

A bundle's components are anonymous templates in its `templates/components/`
directory, referenced by the bundle's Twig namespace plus a colon:
`<twig:Acme:Button type="primary">…</twig:Acme:Button>` maps to
`templates/components/Button.html.twig` in the Acme bundle. Discover namespaces
with `php bin/console debug:twig`.

## Higher-order / wrapper components

A component can wrap another to add markup or behavior. Forward attributes with
the spread operator and forward slot content with `outerBlocks`:

```html+twig
{# templates/components/Modal/Confirm.html.twig #}
{% props confirmText = 'Confirm', cancelText = 'Cancel' %}

<twig:Modal {{ ...attributes.defaults({class: 'modal-confirm'}) }}>
    {{ block(outerBlocks.content) }}
    <div class="modal-actions">
        <button type="button">{{ cancelText }}</button>
        <button type="submit">{{ confirmText }}</button>
    </div>
</twig:Modal>
```

```html+twig
<twig:Modal:Confirm confirmText="Yes, delete it" data-controller="modal">
    This action cannot be undone.
</twig:Modal:Confirm>
```

Key parts: `{{ ...attributes }}` passes attributes through to the wrapped
component; `outerBlocks` forwards the wrapper's content blocks; the wrapper adds
its own props and markup.

## Dynamic template

Resolve the template at render time from a component method by passing
`FromMethod` to the `template:` option:

```php
#[AsTwigComponent(template: new FromMethod('getTemplate'))]
class SearchResults
{
    public string $layout = 'rows';
    public function getTemplate(): string { /* return a template path */ }
}
```

## Testing

Use the `InteractsWithTwigComponents` trait in a `KernelTestCase`:
`mountTwigComponent(name, data)` returns the component instance;
`renderTwigComponent(name, data, content, blocks)` returns a rendered result you
can assert on or crawl (`->crawler()`).
