# XLIFF files and Symfony translations

Companion to [SKILL.md](../SKILL.md). All examples follow Contao's documented
conventions.

## XLIFF (`.xlf`) files

One file per domain, just like the PHP variant. The difference: `category`,
`key`, and (for label/description translations) the pair `index` are joined into
a **single** `trans-unit id`.

Overriding a simple string (`MSC.goBack` in the `default` domain):

```xml
<!-- contao/languages/en/default.xlf -->
<?xml version="1.0" encoding="UTF-8"?>
<xliff version="1.1">
  <file>
    <body>
      <trans-unit id="MSC.goBack">
        <target>Back</target>
      </trans-unit>
    </body>
  </file>
</xliff>
```

Overriding a DCA field's label **and** description — the `.0`/`.1` suffixes are
the pair indices:

```xml
<!-- contao/languages/de/tl_content.xlf -->
<?xml version="1.0" encoding="UTF-8"?>
<xliff version="1.1">
  <file>
    <body>
      <trans-unit id="tl_content.text.0">
        <target>Text-Inhalt</target>
      </trans-unit>
      <trans-unit id="tl_content.text.1">
        <target>Geben Sie hier den Text-Inhalt ein.</target>
      </trans-unit>
    </body>
  </file>
</xliff>
```

`<target>` carries the translated value when you customize/extend an existing
string. (Contao's own canonical English files declare the original value with
`<source>` and a `source-language="en"` file header.)

## Symfony translations (Contao 5.3+)

Provide or override the very same Contao translations through Symfony's
translation component. The Symfony domain is the Contao domain prefixed with
`contao_`, and the **keys stay identical** to the `TL_LANG` / XLIFF keys.

Override `MSC.goBack` (Contao `default` → Symfony `contao_default`):

```yaml
# translations/contao_default.en.yaml
MSC:
    goBack: Return back
```

Labels for your own DCA `tl_foobar` (→ `contao_tl_foobar`). A list maps to the
`[label, description]` pair; `new.1` is the second element of the `new` pair —
the same shape as `$GLOBALS['TL_LANG']['tl_foobar']['new'][1]`:

```yaml
# translations/contao_tl_foobar.en.yaml
tl_foobar:
    new.1: Add a new foobar element
    my_legend: My custom fieldset legend
    my_field:
        - My custom field title
        - My custom field description
```

## Accessing translations from PHP

Legacy load (only needed without the translator; DCA files are not auto-loaded
on every request):

```php
\Contao\System::loadLanguageFile('tl_content', 'de');
// first arg = domain ("language file"), second = language
```

Recommended — inject `TranslatorInterface` and call `trans()` with the
`contao_`-prefixed domain. This also loads the underlying language file, so no
`loadLanguageFile` call is required:

```php
// src/Controller/ExampleController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Contracts\Translation\TranslatorInterface;

#[Route('/app/test', name: ExampleController::class)]
class ExampleController
{
    public function __construct(private TranslatorInterface $translator)
    {
    }

    public function __invoke(): Response
    {
        return new Response($this->translator->trans('MSC.goBack', [], 'contao_default'));
    }
}
```

## Templates

Contao PHP templates expose `trans()` (second/third args default to `[]` and
`contao_default`):

```php
// templates/my_template.html5
<?= $this->trans('MSC.goBack') ?>
<?= $this->trans('XPT.error', [], 'contao_exception') ?>
```

Twig templates use the standard Symfony filter/tag:

```twig
{{ 'MSC.goBack'|trans({}, 'contao_default') }}
```
