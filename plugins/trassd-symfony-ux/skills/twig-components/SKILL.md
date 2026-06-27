---
name: twig-components
description: Build reusable PHP-backed Twig Components with Symfony UX (#[AsTwigComponent], props, mount(), the attributes bag, slots, anonymous components, and the <twig:Name /> HTML syntax). Use when creating or editing a component under src/Twig/Components or templates/components, refactoring duplicated Twig markup into a component, or wiring component props, attributes, slots, or mount hooks.
---

# Twig Components

A Twig Component binds an object to a template so you can render and reuse small
markup "units" (alert, modal, sidebar). It consists of a class plus a template,
and renders via `{{ component('Name', {...}) }}` or the HTML syntax
`<twig:Name ... />`.

## Conventions (where things live)

- **Class:** `src/Twig/Components/<Name>.php`, namespace `App\Twig\Components`,
  marked with `#[AsTwigComponent]`.
- **Template:** `templates/components/<Name>.html.twig`. The component name maps
  to the template path; a class in a subnamespace `Button\Primary` becomes name
  `Button:Primary` → `templates/components/Button/Primary.html.twig` (`:` ↔
  subdirectory).
- The default namespace→directory mapping is set in
  `config/packages/twig_component.yaml` under `twig_component.defaults`. With
  Flex this is auto-created.
- Inspect everything with `php bin/console debug:twig-component` (add a name for
  details). Scaffold with `php bin/console make:twig-component <Name>` if
  MakerBundle is installed.

## The class

```php
#[AsTwigComponent]
class Alert
{
    public string $message;
    public string $type = 'success';
}
```

- Each **public property is a prop**: passed values are set on it (a matching
  setter like `setMessage()` is used if present). Properties are exposed
  directly in the template (`{{ message }}`) and also via `{{ this.message }}`.
- Name is derived from the class; override with `#[AsTwigComponent('alert')]`.
  Override the template with `#[AsTwigComponent(template: '...')]`.
- Components are **services** — constructor autowiring works (e.g. inject a
  repository). Each is registered `shared: false`, so the same component can be
  rendered multiple times with independent state safely.
- Keep components **lazy**: store only what you need on properties and compute
  the rest in getters (`this.products`). For a getter you call more than once,
  use `computed.products` so the result is cached per render.
- **Avoid `readonly` on the component class or on props** — props are assigned
  *after* construction, which a `readonly` property forbids. Use `readonly` only
  for constructor-injected services, or assign the prop in `mount()`.

## Props vs. attributes

Props you pass that *don't* map to a settable property become **HTML
attributes**, collected in the special `attributes` bag. Render them on the root
element and merge defaults:

```html+twig
<div {{ attributes.defaults({class: 'alert alert-' ~ type}) }}>
    {{ message }}
</div>
```

Passed attributes override defaults, **except `class`** which is *prepended*
(defaults first, then passed). See `references/attributes-and-slots.md` for the
full attributes API (`render`, `only`, `without`, `nested`, true/false values).

## mount() and the mount hooks

`mount()` runs once right after instantiation, **before** props are assigned to
properties — so reading `$this->type` inside it gives the *default*, not the
passed value. To use a passed value, declare it as a `mount()` parameter named
after the prop:

```php
public function mount(string $type): void
{
    if ('error' === $type) { /* ... */ }
}
```

A prop captured by a `mount()` parameter is consumed: it is **not** set on a
public property nor added to `attributes`. `mount()` can also accept props that
have no matching property at all (e.g. `bool $isError = false`).

- `#[PreMount]` (method) — receives the raw `array $data` of props *before*
  mounting and returns the array to use; the spot to validate/normalize (e.g.
  with `OptionsResolver`). Multiple hooks order via `#[PreMount(priority: N)]`
  (higher runs earlier).
- `#[PostMount]` (method) — runs *after* data is mounted. May take `array $data`
  of still-unprocessed props, handle/`unset()` some, and return the rest (which
  then become attributes). Order via `#[PostMount(priority: N)]`.

## ExposeInTemplate

`#[ExposeInTemplate]` exposes a private/protected property or a public method as
a bare template variable (`{{ message }}` instead of `{{ this.message }}`).
Properties must be accessible (have a getter); methods must have no required
parameters. Optional first arg renames the variable, and `getter:` picks the
accessor: `#[ExposeInTemplate(name: 'ico', getter: 'fetchIcon')]`. Do **not**
put it on a method you also call via `computed.` — it would run twice.

## HTML syntax — `<twig:Name />`

```html+twig
<twig:Alert message="Hi" withCloseButton />        {# bool true #}
<twig:Alert :user="user.id" />                     {# dynamic: ":" or {{ }} #}
<twig:Alert {{ ...myAttributes }} />               {# spread (Twig >= 3.7) #}
```

- A bare attribute (`withCloseButton`) passes boolean `true`. The string
  `"false"` juggles to `true` — pass real `false` with `:withCloseButton="false"`
  or `="{{ false }}"`.
- Props and root-element attributes mix freely:
  `<twig:Alert message="hi" id="x" />`.

## Slots (passing HTML)

Content between the tags is exposed as a block named `content`:

```html+twig
<twig:Alert>I am <strong>HTML</strong></twig:Alert>
```
```html+twig
<div {{ attributes.defaults({class: 'alert'}) }}>
    {% block content %}{% endblock %}
</div>
```

Define additional **named blocks** (with optional defaults) and fill them with
`<twig:block name="footer">…</twig:block>`. The non-HTML form is
`{% component Alert with {...} %}…{% endcomponent %}`. Full slot/scope rules
(`outerScope`, `outerBlocks`, nested-component caveats) are in
`references/attributes-and-slots.md`.

## Anonymous components (no class)

When a component needs no PHP logic, create only the template under
`templates/components/`; its name is its path. Declare props with the `{% props %}`
tag (required by default, `=` gives a default):

```html+twig
{# templates/components/Button.html.twig #}
{% props icon, type = 'primary' %}
<button {{ attributes.defaults({class: 'btn btn-' ~ type}) }}>
    {% block content %}{% endblock %}
</button>
```

See `references/html-syntax-and-anonymous.md` for naming (`index.html.twig`,
nested directories, third-party bundle namespaces like `<twig:Acme:Button>`) and
higher-order/wrapper components.

## Quick checklist

- Class in `src/Twig/Components`, template in `templates/components`, names match.
- Public properties only for real props; no `readonly` on the class/props.
- Read passed prop values via `mount()` parameters, not `$this->` inside `mount()`.
- Render `{{ attributes }}` (or `attributes.defaults({...})`) on the root element.
- Validate/normalize input in `#[PreMount]`; post-process in `#[PostMount]`.
- Use `computed.` for repeated getter calls; keep components lazy.
