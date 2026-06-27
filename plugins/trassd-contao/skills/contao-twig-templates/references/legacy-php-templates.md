# Legacy PHP templates (`.html5`)

Reference for the legacy engine. **Use this for maintaining existing templates
only** — write new templates in Twig. The legacy engine is planned for removal
in Contao 6. Distilled from the Contao legacy templates docs.

## Instantiation and rendering

Two classes inherit from the abstract `Contao\Template` (via the
`Contao\TemplateInheritance` trait):

- `Contao\FrontendTemplate` — front end (`ce_*`, `mod_*`, `news_*`, `event_*`, …).
- `Contao\BackendTemplate` — back end (mostly `be_*`).

```php
use Contao\FrontendTemplate;

$template = new FrontendTemplate('my_front_end_template'); // -> my_front_end_template.html5
$template->someData = 'foobar';
$buffer = $template->parse();
```

## Template folders (search order)

A template instance searches these locations in order; the first match wins:

| Folder | Purpose |
| --- | --- |
| `templates/<THEME>/` | Theme-specific overrides/extensions for a layout's theme. |
| `templates/` | Global overrides, extensions, and custom templates. |
| `contao/templates/` | Application-specific templates (your app's own elements/modules). |
| `<BUNDLE>/contao/templates/` | Templates shipped by a Contao extension. |

`templates/` (and the theme subfolder) can override **and** simultaneously
extend the same template name — e.g. `templates/fe_page.html5` may still
`$this->extend('fe_page')` to inherit from the core `fe_page`. The same applies
to `contao/templates/`. Inside a bundle you cannot do this; you must extend a
*different* template name.

## Template groups (back end selection)

For a custom template to appear in the back end dropdown, prefix it with the
element/module type:

- Content element: `ce_<type>_<name>` — e.g. `ce_text_ipsum.html5`.
- Module: `mod_<type>_<name>` — e.g. `mod_html_foobar.html5`.
- Form field: `form_<type>_<name>` — e.g. `form_textarea_custom.html5`.
- Sub-items use a single prefix: `news_`, `nav_`, `event_`.

Directories on disk do **not** affect the template name in the legacy engine.

## Inheritance (PHP API)

```php
<?php $this->extend('fe_page'); ?>

<?php $this->block('head'); ?>
    <?php $this->parent(); ?>
    <link rel="stylesheet" href="style_2.css">
<?php $this->endblock(); ?>
```

- `$this->extend('name')` — declare the parent template.
- `$this->block('name')` / `$this->endblock()` — define/override a block.
- `$this->parent()` — keep the parent block's content while adding to it.
- A block can be overridden with empty content to suppress it.

Contao ships base templates `block_searchable` and `block_unsearchable` that
wrap an element/module in a `<div>` with CSS class/ID attributes and provide
`headline` and `content` blocks. `block_unsearchable` additionally wraps content
in `<!-- indexer::stop -->` / `<!-- indexer::continue -->` so it is skipped by
the search indexer. Most core element/module templates extend one of these.

## Template insertion

```php
<?php $this->insert('image-copyright', ['name' => 'Donna Evans']); ?>
<?php $this->insert('template_name', $this->getData()); ?> // pass all current vars
```

## Template data

- Set via property (`$template->foo = 'bar'`, backed by `__set`) or
  `$template->setData([...])`.
- Read in the template with `<?= $this->foo ?>`.
- Callables assigned to a variable are invoked via `__call`:
  `$template->getHash = fn($s) => substr(md5($s), 0, 8);` then
  `<?= $this->getHash('foobar') ?>`.
- Inspect data with `<?php $this->dumpTemplateVars() ?>` or `<?php dump($this) ?>`.

## Lazy variables

Assign a closure and it runs only when first accessed in the template:

```php
$template->foo = function (): string { /* … */ return $result; };
```

Wrap with `Contao\Template::once(...)` to compute once and reuse the result on
subsequent accesses (as `$this->text`/`$this->hasText` do for news).

## Overriding a legacy template with Twig (interop)

Give a Twig file the **same name** as the legacy template but a `.html.twig`
extension; when both exist the legacy one is ignored. Twig may `extends` a legacy
template (the reverse is not possible):

```twig
{% extends "@Contao/fe_page.html5" %}

{% block head %}
    {{ parent() }}
    <style>.thing { color: orange; }</style>
{% endblock %}
```

Legacy template data is transformed into Twig context by the
`@contao.twig.interop.context_factory` service (the `Contao\CoreBundle\Twig\Interop`
namespace). Callables become objects exposing `invoke()`:

```twig
{{ normalValue }}
{{ fooFunction.invoke('bar') }}
{{ lazyArray.invoke()|join(', ') }}
```

Prefer real objects over callables for new code; the interop layer is slated for
removal with the rest of the legacy engine.

> Note: in current Contao core, `fe_page` ships as `fe_page.html.twig` rather
> than `.html5`; the `.html5` example above reflects the documented interop
> mechanism and still applies to any remaining legacy template.
