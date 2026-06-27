# Fragment controllers and dynamic rendering

How content elements and front end modules render Twig templates. Distilled from
the Contao templates quick-reference docs.

## Content element controller

A content element controller extends `AbstractContentElementController`, carries
the `#[AsContentElement(...)]` attribute, and renders a
`content_element/<type>` template. The `getResponse()` method receives a
`FragmentTemplate`, the model, and the request:

```php
<?php

namespace App\Controller;

use Contao\ContentModel;
use Contao\CoreBundle\Controller\ContentElement\AbstractContentElementController;
use Contao\CoreBundle\DependencyInjection\Attribute\AsContentElement;
use Contao\CoreBundle\Twig\FragmentTemplate;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

#[AsContentElement(category: 'food')]
class FruitSaladController extends AbstractContentElementController
{
    protected function getResponse(FragmentTemplate $template, ContentModel $model, Request $request): Response
    {
        $template->set('fruits', ['acai', 'blackberry', 'currant']);

        return $template->getResponse();
    }
}
```

## Front end module controller

A module works the same way but uses:

- base class `AbstractFrontendModuleController`,
- attribute `#[AsFrontendModule(...)]`,
- parent template `frontend_module/_base.html.twig`,
- DCA target `tl_module` (instead of `tl_content`).

## Base template + variant

The default template extends the `_base` component and overrides `content`:

```twig
{# content_element/fruit_salad.html.twig #}
{% extends "@Contao/content_element/_base.html.twig" %}

{% block content %}
    <p>We put fruit like {{ fruits|join(', ') }} in our tropical salads.</p>
{% endblock %}
```

A variant lives under a directory named after the base template, so the back end
offers it in the template dropdown:

```twig
{# content_element/fruit_salad/random.html.twig #}
{% extends "@Contao/content_element/_base.html.twig" %}
{% use "@Contao/component/_list.html.twig" %}

{% block content %}
    {% with {items: fruits, randomize_order: true} %}
        {{ block('list_component') }}
    {% endwith %}
{% endblock %}

{% block list_item %}
    <span class="fruit--{{ item }}">{{ item|capitalize }}</span>
{% endblock %}
```

To allow selecting a variant in the back end, the element's DCA palette must
include the `customTpl` field, e.g.:

```php
$GLOBALS['TL_DCA']['tl_content']['palettes']['fruit_salad'] =
    '{type_legend},type,headline;'
    . '{template_legend:collapsed},customTpl';
```

## Choosing the template at runtime

Two equivalent ways inside `getResponse()`:

**Override the inferred name on the `FragmentTemplate`:**

```php
$template->set('foo', 'bar');
$template->setName('@Contao/content_element/custom/thing.html.twig');

return $template->getResponse();
```

**Call `render()` yourself**, merging the pre-built data (type, headline, CSS
classes) via `$template->getData()`:

```php
$parameters = ['foo' => 'bar', ...$template->getData()];

return $this->render('@Contao/content_element/amazing_fruit.html.twig', $parameters);
```

`$template->getResponse()` internally calls `$this->render(null, $template->getData())`,
where `null` resolves to the inferred default template name — which is why the
two approaches are equivalent.

## Adjusting the shared base template

To affect every content element, override `content_element/_base.html.twig`.
`HtmlAttributes` (built via `attrs()`) lets you add an attribute by extending the
base alone — no block override needed:

```twig
{% extends "@Contao/content_element/_base.html.twig" %}

{% set attributes = attrs().set('data-element', data.id).mergeWith(attributes|default) %}
```

## Building template option lists

When your own code needs a list of matching templates (e.g. for a DCA options
callback), inject the `contao.twig.finder_factory` service and use its fluent
finder rather than walking the hierarchy by hand:

```php
$finder = $this->finderFactory->create()
    ->identifier('foo/bar')
    ->extension('json.twig')
    ->withVariants();

$options = $finder->asTemplateOptions();
```

## Contao 4.13 vs 5 caveat

The auto-generated template name from a fragment controller differs between
versions: a `FooController` content element maps to `ce_foo` in 4.13 but
`content_element/foo` in Contao 5. To use the new structure on 4.13, set the
identifier explicitly in the attribute:

```php
#[AsContentElement(category: 'bar', template: 'content_element/foo')]
```
