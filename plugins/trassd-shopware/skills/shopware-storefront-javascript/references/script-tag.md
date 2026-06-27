# Adding JavaScript as a plain `<script>` tag

Normally you add JS to `main.js` so it is compiled into the Storefront bundle.
Use a `<script>` tag instead only when you need an external/CDN library or a
script that must live in the page HTML.

## Recommended location: the head block

Extend `@Storefront/storefront/layout/meta.html.twig` and add your script inside
the `layout_head_javascript_hmr_mode` block so it sits alongside the default
Storefront scripts:

```twig
{# <plugin root>/src/Resources/views/storefront/layout/meta.html.twig #}
{% sw_extends '@Storefront/storefront/layout/meta.html.twig' %}

{% block layout_head_javascript_hmr_mode %}
    {{ parent() }}   {# renders the Storefront JS — MUST be kept #}
    <script src="https://unpkg.com/isotope-layout@3/dist/isotope.pkgd.min.js" defer></script>
{% endblock %}
```

> Danger: if you override `layout_head_javascript_hmr_mode` you must keep
> `{{ parent() }}`. Omitting it removes the core Storefront JavaScript and breaks
> the Storefront. Only drop it if you explicitly intend to replace core JS.

## Conditional scripts

Wrap the tag in a Twig condition to render it only when needed:

```twig
{% block layout_head_javascript_hmr_mode %}
    {{ parent() }}
    {% if someCondition %}
        <script src="https://unpkg.com/isotope-layout@3/dist/isotope.pkgd.min.js" defer></script>
    {% endif %}
{% endblock %}
```

## Script order

- If the Storefront JS does **not** need your library, place the script **after**
  the Storefront JavaScript.
- If the Storefront JS **does** need access to it, place the script **before** the
  Storefront JavaScript. Be aware that a non-async script placed before it will
  postpone the Storefront JS execution.

## Loading behavior

- Prefer `defer` so the script runs after the document is parsed.
- Use `async` only if the library's own docs require it.
- Avoid external `<script src>` without `defer`/`async` — it blocks rendering and
  hurts performance.

## Alternative locations

You can place a `<script>` at other Twig block locations, e.g. near the body via
the `base_body_script` block in
`@Storefront/storefront/base.html.twig`. Do this only when there is a technical
reason (for example a library that mandates a specific position in the HTML).
