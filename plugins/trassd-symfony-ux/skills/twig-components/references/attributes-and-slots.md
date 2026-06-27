# Attributes bag & slots

## The attributes bag

Any prop that can't be mounted on the component lands in the `attributes`
object, available in every component template. Render it on an element:

```html+twig
<div {{ attributes }}>My Component!</div>
```

Rendering `<twig:MyComponent class="foo" style="color: red" />` yields
`<div class="foo" style="color:red">`.

- Attribute value `true` renders just the name (`:autofocus="true"` →
  `autofocus`).
- Attribute value `false` omits the attribute entirely.
- The variable name can be changed: `#[AsTwigComponent(attributesVar: '_attributes')]`.

### defaults() and merging

```html+twig
<button {{ attributes.defaults({class: 'bar', type: 'button'}) }}>Save</button>
```

Passed attributes override the defaults — **except `class`**, where defaults are
**prepended**. Rendering with `{class: 'foo', type: 'submit'}` gives
`class="bar foo" type="submit"`.

### render(), only(), without()

- `attributes.render('style')` outputs a single attribute's value, so you can
  splice it into custom markup. **Call `render()` before `{{ attributes }}`**,
  and still render the remaining `{{ attributes }}`, or the attribute is emitted
  twice.
- `attributes.only('class')` keeps only the listed attributes.
- `attributes.without('class')` drops the listed attributes.

### Nested attributes

Route attributes to descendant elements with `attributes.nested('name')`; pass
them with a `name:` prefix. Nesting is recursive.

```html+twig
{# Dialog.html.twig #}
<div {{ attributes }}>
    <div {{ attributes.nested('title') }}>{% block title %}{% endblock %}</div>
    <div {{ attributes.nested('body') }}>{% block content %}{% endblock %}</div>
</div>
```
```html+twig
<twig:Dialog class="foo" title:class="bar" body:class="baz">Content</twig:Dialog>
{# row:label:class="..." nests further #}
```

### Stimulus

`{{ attributes.defaults(stimulus_controller('my-controller', {someValue: 'foo'})) }}`
attaches a Stimulus controller to the root element (requires
`symfony/stimulus-bundle`).

## Slots / passing HTML

The content between `<twig:Name>…</twig:Name>` becomes the `content` block.
Define more named blocks (with default content) and fill them:

```html+twig
{# Alert.html.twig #}
<div class="alert alert-{{ type }}">
    {% block content %}{% endblock %}
    {% block footer %}<div>Default footer</div>{% endblock %}
</div>
```
```html+twig
<twig:Alert type="success">
    <div>Body</div>
    <twig:block name="footer">
        {{ parent() }}
        <button>Claim</button>
    </twig:block>
</twig:Alert>
```

Non-HTML equivalent:

```html+twig
{% component Alert with {type: 'success'} %}
    {% block content %}<div>Body</div>{% endblock %}
    {% block footer %}…{% endblock %}
{% endcomponent %}
```

### Scope inside slots

Content inside a `<twig:…>` tag behaves like a template that *extends* the
component's template:

- `this` refers to the **inner** (current) component.
- All variables from the outer template are available too, but inner-component
  variables with the same name win (the variables are merged).
- Reach the parent's local context with `outerScope` (e.g.
  `outerScope.this.someProp`, `outerScope.name`); chain `outerScope.outerScope.…`
  for grandparents. `outerScope` only applies once you are *inside* the embedded
  component.
- Blocks from the outer template are reachable via `outerBlocks`, e.g.
  `block(outerBlocks.call_to_action)` — useful to forward a `content` block into
  a wrapped component.

### Macros inside components

Import macros by full template path, not `_self`:
`{% from 'path/to/template.html.twig' import my_macro %}`.

### Nested components: don't mix syntaxes

When nesting components, use one syntax consistently. Mixing the HTML
`<twig:…>` form with `{% block %}` (Twig form) in the same nesting breaks; use
`<twig:block name="footer">` with HTML tags, or `{% component %}` with
`{% block %}` throughout.

## provide() / inject()

Share a value from an ancestor to deep descendants without threading props:

```html+twig
{# parent #}
{% do provide('inputOtp.maxLength', maxLength) %}
```
```html+twig
{# any descendant #}
{% set maxLength = inject('inputOtp.maxLength', 4) %}  {# 4 = fallback #}
```

Rules: values flow top-down only; `inject()` walks ancestors nearest-first and
skips the current component; provides are dropped once the parent finishes
rendering (siblings never share); call `provide()` at the top of the parent
before `{% block content %}`; last `provide()` of a key wins; prefix keys with
the component name to avoid collisions.
