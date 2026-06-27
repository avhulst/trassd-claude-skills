---
name: contao-fragment-controllers
description: >-
  Build content elements, front end modules, and back end modules as fragment
  controllers — the #[AsContentElement] / #[AsFrontendModule] attributes, the
  AbstractContentElementController / AbstractFrontendModuleController base
  classes, the getResponse() template pattern, the matching DCA palette, and
  template naming conventions. Use when adding or editing a Contao content
  element, front end module, or back end module (files under
  src/Controller/ContentElement, src/Controller/FrontendModule, a tl_content /
  tl_module palette, or $GLOBALS['BE_MOD']).
---

# Contao Fragment Controllers

Content elements and front end modules in Contao are **fragment controllers** —
Symfony controllers rendered as *sub requests* (partials of a page), so they have
no route but still receive a `Request` and return a `Response`. Each is a
service tagged so the Contao framework (`ContentProxy` / `ModuleProxy`) makes it
behave like a classic Contao element/module. Available since Contao 4.5.

Three things are always required: **a controller class**, **a DCA palette**, and
**a template**. Add a translation label so the back end shows a readable name.

## Content elements

Extend `AbstractContentElementController` and implement `getResponse()`. Tag it
with `#[AsContentElement]`. The element renders into article content of pages,
news, events, etc. (`tl_content`).

```php
namespace App\Controller\ContentElement;

use Contao\ContentModel;
use Contao\CoreBundle\Controller\ContentElement\AbstractContentElementController;
use Contao\CoreBundle\DependencyInjection\Attribute\AsContentElement;
use Contao\CoreBundle\Twig\FragmentTemplate;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

#[AsContentElement(category: 'texts')]
class MyTextElementController extends AbstractContentElementController
{
    protected function getResponse(FragmentTemplate $template, ContentModel $model, Request $request): Response
    {
        $template->set('text', $model->text);

        return $template->getResponse();
    }
}
```

## Front end modules

Identical pattern, but extend `AbstractFrontendModuleController`, tag with
`#[AsFrontendModule]`, and receive a `ModuleModel` (`tl_module`). Use modules
for reusable functionality placed in layout sections (lists, navigation, forms).

```php
namespace App\Controller\FrontendModule;

use Contao\CoreBundle\Controller\FrontendModule\AbstractFrontendModuleController;
use Contao\CoreBundle\DependencyInjection\Attribute\AsFrontendModule;
use Contao\CoreBundle\Twig\FragmentTemplate;
use Contao\ModuleModel;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

#[AsFrontendModule(category: 'miscellaneous')]
class ExampleModuleController extends AbstractFrontendModuleController
{
    protected function getResponse(FragmentTemplate $template, ModuleModel $model, Request $request): Response
    {
        $template->set('action', $request->getUri());

        return $template->getResponse();
    }
}
```

## The getResponse() / template pattern

- Signature is `getResponse(FragmentTemplate $template, <ContentModel|ModuleModel> $model, Request $request): Response`.
- A `FragmentTemplate` for the resolved template is created automatically and
  passed in. Push variables with `$template->set('name', $value)` (or
  `$template->setData([...])`), then **return `$template->getResponse()`** — this
  renders the Twig template and returns it as the `Response`.
- Read DCA field values from `$model` (e.g. `$model->text`, `$model->jumpTo`).
- `$this->getPageModel()` returns the current `\Contao\PageModel`.
- `$this->isBackendScope($request)` lets a controller return an empty `Response()`
  in the back end (e.g. for wrapper elements).
- Throw `Contao\CoreBundle\Exception\RedirectResponseException` to redirect.

## Attribute / service-tag options

`#[AsContentElement]` (tag `contao.content_element`) and `#[AsFrontendModule]`
(tag `contao.frontend_module`) accept the same options. Define only what you
need — defaults are sensible.

| Option | Meaning |
| --- | --- |
| `type` | Identifier used for the palette key and template name. **Default:** class name PascalCase → snake_case with a trailing `Controller` removed (e.g. `MyTextElementController` → `my_text_element`). |
| `category` | Option group in the back end type dropdown. **Default:** `miscellaneous`. |
| `template` | Override the generated template name. |
| `renderer` | `forward` (default), `inline`, or `esi` — controls sub-request rendering/caching. |
| `method` | Method to invoke (defaults to `getResponse`/`__invoke`). |
| `priority` | Resolution priority. |

Registration can also be done via the `@ContentElement` / `@FrontendModule`
annotation or the raw service tag in `config/services.yaml`; the PHP attribute is
preferred. Controllers that **extend a legacy Contao class** (instead of an
`Abstract*Controller`) are not auto-registered and must be declared as services
manually. See [references/registration.md](references/registration.md).

## The matching DCA palette

The palette key equals the resolved **type**. Without a palette the element/module
cannot be configured in the back end.

```php
// contao/dca/tl_content.php  (content element)
$GLOBALS['TL_DCA']['tl_content']['palettes']['my_text_element'] = '
    {type_legend},type,headline;
    {text_legend},text
';

// contao/dca/tl_module.php  (front end module)
$GLOBALS['TL_DCA']['tl_module']['palettes']['example_module'] = '
    {title_legend},name,headline,type;
    {redirect_legend},jumpTo;
';
```

Add the back-end label (omitting it shows the raw type):

```yaml
# translations/contao_default.en.yaml — content elements
CTE:
    my_text_element:
        - My Text Element
        - A short description.
```
```yaml
# translations/contao_modules.en.yaml — front end modules
FMD:
    example_module:
        - My Module
        - A short description.
```

## Templates: naming convention

The template name is derived from the **type** unless overridden via the
`template` option.

- **Twig (recommended):** `templates/content_element/<type>.html.twig` and
  `templates/frontend_module/<type>.html.twig`. Extend the matching base and
  fill the `content` block:

  ```twig
  {# templates/content_element/my_text_element.html.twig #}
  {% extends "@Contao/content_element/_base.html.twig" %}

  {% block content %}
      {{ text }}
  {% endblock %}
  ```

- **Legacy PHP templates** use the `ce_*` (content element) and `mod_*` (front
  end module) prefixes, e.g. `contao/templates/mod_example_module.html5`. Prefer
  Twig for new code.

For nested fragments, parent-table rendering, and legacy details see
[references/templates-and-advanced.md](references/templates-and-advanced.md).

## Back end modules

A back end module is a **back end navigation entry**, not a fragment controller.
Register it in `$GLOBALS['BE_MOD']` under a category, listing the DCA `tables` it
manages:

```php
// contao/config/config.php
$GLOBALS['BE_MOD']['content']['my_module'] = [
    'tables' => ['tl_my_module'],
];
```

Clicking the module loads the DCA of the first/relevant table. Supported keys:
`tables`, `stylesheet`, `javascript`, `callback`, `disablePermissionChecks`,
`hideInNavigation`, and custom `<key>` callbacks triggered via `&key=…` query
params. Label it via a `modules.xlf` file (`MOD.<module>.0` / `.1`). For
DI-based output prefer custom back end routes over the simple `callback` class.
See [references/back-end-modules.md](references/back-end-modules.md), including
how to attach `tl_content` editing to your own table.

## Checklist

- [ ] Controller extends `AbstractContentElementController` /
      `AbstractFrontendModuleController` and implements `getResponse()`.
- [ ] Tagged with `#[AsContentElement]` / `#[AsFrontendModule]` (set `category`).
- [ ] `getResponse()` pushes data with `$template->set()` and returns
      `$template->getResponse()`.
- [ ] DCA palette under the **type** key exists (`tl_content` / `tl_module`).
- [ ] Template at `content_element/<type>` or `frontend_module/<type>`
      (or `ce_*` / `mod_*` legacy) exists.
- [ ] Back-end label added (`CTE` / `FMD`).
- [ ] Legacy-class controllers registered manually as services.
- [ ] Back end module registered in `$GLOBALS['BE_MOD']` with its `tables`.
