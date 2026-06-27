---
name: contao-twig-templates
description: >-
  Build and customize Contao Twig templates — the template hierarchy,
  _index/component conventions, overriding core templates, and the modern vs
  legacy template systems. Triggers when adding or overriding templates under
  contao/templates or a bundle's templates/ directory, working with the @Contao
  namespace, extending content element / front end module templates, or
  migrating .html5 PHP templates to .html.twig.
---

# Contao Twig templates

Contao's modern template system is built on Twig. Native Twig support arrived in
Contao 4.12; from Contao 5 most content elements are Twig-only, and the legacy
PHP (`.html5`) engine is planned for removal in Contao 6. **Write new templates
in Twig.** Only touch the legacy engine to maintain existing `.html5` files.

## When to use this skill

- Adding a template for a content element, front end module, or your own code.
- Overriding or extending a core/bundle template (e.g. tweaking
  `content_element/text`, adding a `<style>` to `fe_page`).
- Creating template variants that editors can pick in the back end.
- Deciding between the `@Contao` namespace and a plain bundle namespace.
- Migrating a `.html5` PHP template to `.html.twig`.

## The `@Contao` managed namespace

Twig identifies templates by a *logical name*: `@Namespace/identifier.html.twig`.
The *identifier* is everything after the namespace minus the file extension
(e.g. `content_element/text`).

Contao's `ContaoFilesystemLoader` adds every template from a Contao template
directory to a shared **managed `@Contao` namespace**, plus a per-source
namespace (`@Contao_App`, `@Contao_Global`, `@Contao_<Bundle>`,
`@Contao_Theme_<slug>`).

**Always use `@Contao` for anything in the Contao ecosystem.** Standard Symfony
bundle namespaces (and the `templates/bundles/<Bundle>` override trick) require
every overriding party to know each other's namespaces — only the first wins.
`@Contao` solves this. Use a plain bundle namespace only when external
adjustments are explicitly *not* intended.

## How overriding works (the hierarchy)

There is no single file per logical name — there is a **hierarchy**. The loader
ranks sources roughly: application `templates/` (global) and `contao/templates`
(app) first, then bundles in inverse load order (loaded later = wins earlier).
Themes are a runtime overlay on top (see below).

At **compile time**, Contao rewrites `extends`/`include`/`embed`/`use` references
that target `@Contao` to the *next more specific* template in the hierarchy:

```twig
{% extends "@Contao/content_element/text.html.twig" %}
{# compiled to e.g. @Contao_ContaoCoreBundle/content_element/text.html.twig #}
```

**To override a core template, create a template with the same identifier in a
higher-priority directory** (your app's `contao/templates` or global
`templates/`). To *also* extend the original, `{% extends "@Contao/<same name>" %}`
— the rewrite resolves to the version below yours, so multiple independent
extensions stack cleanly without knowing about each other.

Use the `debug:contao-twig` command to inspect the built hierarchy.

## Extend / block / parent() and use

Override only the blocks you care about and call `parent()` to keep inherited
content:

```twig
{% extends "@Contao/content_element/_base.html.twig" %}

{% block content %}
    {{ parent() }}
    <p>Extra content.</p>
{% endblock %}
```

`{% use %}` pulls blocks from a **component** template (a partial, named with a
leading underscore, e.g. `component/_list.html.twig`) so you can call them as
blocks and override their inner blocks:

```twig
{% use "@Contao/component/_list.html.twig" %}

{% with {items: fruits, randomize_order: true} %}
    {{ block('list_component') }}
{% endwith %}

{% block list_item %}<span>{{ item|capitalize }}</span>{% endblock %}
```

## Naming, structure, and variants

- The directory path is part of the identifier:
  `content_element/text.html.twig` → identifier `content_element/text`. Group by
  category (`content_element/`, `frontend_module/`, `component/`).
- Include both filetype and `.twig`: `foo.html.twig`, `icon.svg.twig`. Avoid
  extra dots. Each identifier must have a unique extension.
- **Twig root:** only the global `templates/` and theme dirs are implicit roots
  today (until Contao 6). For other dirs (a bundle, `contao/templates`), drop a
  `.twig-root` marker file to mark the naming root; directory layers above it are
  ignored in the identifier.
- **Variant templates:** place a template in a directory matching the base
  template's name (`content_element/text/highlight.html.twig`) and extend the
  base. It is auto-discovered and offered in the back end template dropdown.
- **Themes** are a *runtime* representation of an otherwise-existing template —
  they are **not** part of the hierarchy and never appear in dropdowns; create a
  selectable non-theme base first.

## Output encoding (security)

Twig uses **output encoding** and auto-escapes per file extension (`.html.twig`
→ HTML). Choose escaping by context with `|e('css')`, `|e('html')`, etc. Use
`|raw` **only** on trusted HTML (e.g. sanitized rich-text) — misuse is an XSS
risk. Contao's escaper prevents double encoding inside `@Contao*` namespaces.

## Fragment controllers and rendering

Content element / front end module controllers extend
`AbstractContentElementController` / `AbstractFrontendModuleController` and
receive a `FragmentTemplate`. Set data and return a response; the template name
is inferred from the type (`content_element/<type>` in Contao 5):

```php
$template->set('fruits', [...]);
return $template->getResponse();
```

Override the inferred name with `$template->setName('@Contao/...')`, or render
directly with `$this->render('@Contao/...', $params)`. A template finder is
available via the `contao.twig.finder_factory` service for building template
option lists. See [references/fragment-controllers.md](references/fragment-controllers.md).

## Legacy PHP templates (`.html5`) — maintenance only

The legacy engine differs fundamentally and should not be used for new code:

- Files are `<name>.html5`, mostly HTML + raw PHP, using **input** encoding.
- Rendered via `Contao\FrontendTemplate` / `Contao\BackendTemplate`; data set
  with `$this->foo = ...` and output with `<?= $this->foo ?>`.
- Inheritance uses PHP calls: `$this->extend('block_unsearchable')`,
  `$this->block('content')` / `$this->endblock()`, `$this->parent()`,
  `$this->insert('name', [...])`. Naming is **prefix**-based (`ce_`, `mod_`,
  `form_`, `news_`, `nav_`, `event_`, `be_`) — directories don't affect the name.

**Interop:** you may override/extend a legacy template *with* a Twig template by
giving it the same name with a `.html.twig` extension (e.g. extend
`@Contao/fe_page`). When both exist, the legacy one is ignored. You cannot go
the other way (PHP extending Twig). See
[references/legacy-php-templates.md](references/legacy-php-templates.md) for the
PHP API and migration notes.

## Quick rules

- New templates: Twig, `@Contao` namespace, structured by directory.
- Override = same identifier higher in the hierarchy; `extends` + `block` +
  `parent()` to build on the original.
- Components are `_`-prefixed partials consumed via `use`.
- Variants live under a directory named after their base template.
- Never add `|raw` to untrusted data.
- Don't write new `.html5` templates; migrate them to `.html.twig`.
