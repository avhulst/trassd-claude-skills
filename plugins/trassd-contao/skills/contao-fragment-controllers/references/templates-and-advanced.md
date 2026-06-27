# Templates, nested fragments & rendering elements for your own table

## Template resolution

| Kind | Twig path | Legacy prefix |
| --- | --- | --- |
| Content element | `templates/content_element/<type>.html.twig` | `ce_*` (e.g. `ce_my_element.html5`) |
| Front end module | `templates/frontend_module/<type>.html.twig` | `mod_*` (e.g. `mod_example.html5`) |

`<type>` is the resolved type (auto-derived from the class name or set via the
`type` option). Override with the attribute/tag `template` option. Twig templates
extend the corresponding `_base.html.twig` and override the `content` block:

```twig
{% extends "@Contao/content_element/_base.html.twig" %}
{% block content %}
    {{ text }}
{% endblock %}
```

A `FragmentTemplate` instance is created for the resolved template automatically
and handed to `getResponse()`; returning `$template->getResponse()` renders it.

## Nested fragments (Contao 5.3+)

Let a content element contain other content elements (an alternative to the older
wrapper start/stop elements). Enable via the `nestedFragments` option:

```php
#[AsContentElement(nestedFragments: true)]
class ExampleElementController extends AbstractContentElementController { /* … */ }
```

Restrict children to specific types with `allowedTypes`:

```php
#[AsContentElement(nestedFragments: ['allowedTypes' => ['image', 'video']])]
```

`AbstractContentElementController` passes the available children to the template
as `nested_fragments` (a collection of `ContentElementReference`), rendered with
the `content_element()` Twig function:

```twig
{% extends "@Contao/content_element/_base.html.twig" %}
{% block content %}
    {% for fragment in nested_fragments %}
        {{ content_element(fragment) }}
    {% endfor %}
{% endblock %}
```

## Wrapper elements (legacy)

Before nested fragments, "wrapper" elements (`start`/`stop`, also `single` /
`separator`) opened and closed markup around a group of elements. Register them
in `$GLOBALS['TL_WRAPPERS']`:

```php
// contao/config/config.php
$GLOBALS['TL_WRAPPERS']['start'][] = 'my_start_element';
$GLOBALS['TL_WRAPPERS']['stop'][]  = 'my_stop_element';
```

A single controller can carry multiple attributes and should emit different (or
empty) markup in the back end so the wrapper does not break the back end view:

```php
#[AsContentElement('my_start_element')]
#[AsContentElement('my_stop_element')]
class MyWrapperElementController extends AbstractContentElementController
{
    protected function getResponse(FragmentTemplate $template, ContentModel $model, Request $request): Response
    {
        if ($this->isBackendScope($request)) {
            return new Response();
        }

        return $template->getResponse();
    }
}
```

## Reading page data

If the controller extends an `Abstract*Controller` (or `AbstractFragmentController`),
`$this->getPageModel()` returns the current `\Contao\PageModel`:

```php
$page = $this->getPageModel();
$template->set('rootTitle', $page->rootPageTitle ?: $page->rootTitle);
```

## Rendering content elements for your own parent table

To render `tl_content` records belonging to a custom parent record, fetch and
render them inside a module, ideally lazily via a closure so the work happens
only if the template accesses the variable:

```php
$template->set('content', function () use ($request, $parentId): ?string {
    $elements = ContentModel::findPublishedByPidAndTable($parentId, 'tl_example');
    if (null === $elements) {
        return null;
    }

    $section = $request->attributes->get('section', 'main');
    $content = '';
    foreach ($elements as $element) {
        $content .= Controller::getContentElement($element->id, $section);
    }

    return $content;
});
```

Key APIs: `\Contao\ContentModel::findPublishedByPidAndTable($pid, $ptable)`
returns the models; `\Contao\Controller::getContentElement($id, $section)` renders
one. The layout section lives in the `section` request attribute.

See `references/back-end-modules.md` for the DCA / `ptable` wiring that lets a
back end module edit `tl_content` for a custom parent table.
