---
name: contao-translations
description: Add and use translations in Contao — the XLIFF/PHP language files, the contao/languages structure, and the translator. Triggers when adding language files under contao/languages or translating labels in a Contao extension.
---

# Contao Translations

Contao predates its Symfony foundation, so it ships **two translation systems**.
Know which one applies before adding files.

| System | What it is | When it applies |
| --- | --- | --- |
| **Legacy `$GLOBALS['TL_LANG']`** | Arrays loaded from `contao/languages/<lang>/*.php` or `*.xlf` | DCA field labels, back end modules, front/back end UI strings — anything Contao reads from `TL_LANG`. Always available. |
| **Symfony translator** | The `contao_*` translation domains + `trans()` | Contao **5.3+** only. Provide/override the same Contao translations via `translations/contao_<domain>.<lang>.yaml`, or translate in controllers/templates. |

Both write into the **same** logical translation set: the Symfony domain
`contao_default` is the legacy `default` domain, keys are identical. Pick one
mechanism per translation; do not duplicate a key in both.

## Where files live

- Application: `contao/languages/<lang>/`
- Extension: `Resources/contao/languages/<lang>/`
- Symfony YAML overrides: `translations/contao_<domain>.<lang>.yaml`

`<lang>` is the language code directory: ISO 639 (`de`) or POSIX locale (`de_AT`)
for the front end; back end languages are limited to those in
`contao.intl.enabled_locales`. **`en` is always the fallback** when a string is
missing in the current language — author English first.

## Structure: Language » Domain » Category » Key » Label

- **Domain** = one file per domain inside the language directory.
- **Translation ID** = `category` + `key`, e.g. `MSC.goBack`.

```php
// contao/languages/en/default.php
$GLOBALS['TL_LANG']['MSC']['goBack'] = 'Go back';
```

Many places (DCA fields, back end modules) expect a **[label, description] pair**:

```php
$GLOBALS['TL_LANG']['tl_module']['headline'] = [
    'Text',
    'You can use HTML tags to format the text.',
];
```

### Standard domains

`default` (general front/back end), `exception` (error messages),
`explain` (help-wizard content), `modules` (module labels), `countries`,
`languages`. Plus **one domain per Data Container**, named after it — e.g.
`tl_content` translations go in `tl_content.xlf`/`tl_content.php`.

Common categories: `MSC` (miscellaneous), `ERR` (errors), `MOD`/`FMD` (back end
/ front end module labels in `modules`), `XPT` (in `exception`), `CTE`/`PTY`.
See the full domain/category tables in the framework doc when picking placement.
When adding your *own* translations, only the domain choice matters; categories
and keys are free. Use `default` for cross-cutting strings.

## XLIFF instead of PHP

Same one-file-per-domain rule, but category + key (+ pair index) collapse into a
single `trans-unit id`, e.g. `MSC.goBack` or `tl_content.text.0` /
`tl_content.text.1`. See [references/xliff-and-symfony.md](references/xliff-and-symfony.md).

## Accessing translations

Legacy `TL_LANG` is populated for the current request. DCA (`tl_*`)
translations load only when needed in the back end — load on demand:

```php
\Contao\System::loadLanguageFile('tl_content', 'de');
```

**Prefer the Symfony translator** — prepend the Contao domain with `contao_`; it
loads the language file automatically (no `loadLanguageFile` call needed):

```php
$translator->trans('MSC.goBack', [], 'contao_default');
```

In **Contao PHP templates** (`*.html5`): `<?= $this->trans('MSC.goBack') ?>`
(defaults to the `contao_default` domain). In **Twig**:

```twig
{{ 'MSC.goBack'|trans({}, 'contao_default') }}
```

Detailed translator/YAML examples: [references/xliff-and-symfony.md](references/xliff-and-symfony.md).

## Rules

- Author the English (`en`) strings — they are the fallback for every language.
- One translation, one mechanism (legacy file **or** Symfony YAML), not both.
- Match the domain exactly; the Symfony domain is the Contao domain + `contao_`.
- For DCA/module labels remember the `[label, description]` pair shape.
- Don't hand-load DCA language files when using the translator — `contao_*`
  domains load them for you.
