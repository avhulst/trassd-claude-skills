# Registration variants & extending legacy classes

The PHP attribute is the recommended way to register a fragment controller, but
the underlying mechanism is a tagged service. All three styles produce the same
result; pick one.

## PHP attribute (preferred)

```php
#[AsContentElement(
    type: 'example_element',
    category: 'texts',
    template: 'content_element/example',
    renderer: 'forward',
    method: '__invoke',
    nestedFragments: false,
    priority: 100,
)]
class ExampleElementController extends AbstractContentElementController { /* … */ }
```

```php
#[AsFrontendModule(
    type: 'example',
    category: 'miscellaneous',
    template: 'frontend_module/example',
    renderer: 'forward',
    method: '__invoke',
    priority: 100,
)]
class ExampleModuleController extends AbstractFrontendModuleController { /* … */ }
```

Only `category` is commonly set; leave the rest at defaults unless needed.

## Annotation

Usable on the class (if invokable or extending an `Abstract*Controller`) or on a
specific method:

```php
/**
 * @ContentElement(category="texts")
 */
class ExampleElementController extends AbstractContentElementController { /* … */ }

/**
 * @FrontendModule("example", category="miscellaneous", template="mod_example", renderer="forward", method="__invoke")
 */
class ExampleModuleController extends AbstractFrontendModuleController { /* … */ }
```

## YAML service tag

```yaml
# config/services.yaml
services:
    App\Controller\ContentElement\ExampleElementController:
        tags:
            - { name: contao.content_element, category: texts, template: ce_example, renderer: forward, method: __invoke }

    App\Controller\FrontendModule\ExampleModuleController:
        tags:
            - { name: contao.frontend_module, category: miscellaneous, template: mod_example }
```

The tag `name` must be exactly `contao.content_element` or
`contao.frontend_module`.

## Extending a legacy Contao class

Instead of an `Abstract*Controller` you may extend an existing legacy element/
module class and use it as a controller. **These are NOT auto-registered** — you
must declare them as services in your own `config/services.yaml`.

```php
namespace App\Controller\FrontendModule;

use Contao\CoreBundle\DependencyInjection\Attribute\AsFrontendModule;
use Contao\ModuleModel;
use Contao\ModuleNewsList;
use Symfony\Component\HttpFoundation\Response;

#[AsFrontendModule(category: 'news', priority: 1)]
class AppExampleController extends ModuleNewsList
{
    public function __construct() {}

    public function __invoke(ModuleModel $model, string $section): Response
    {
        parent::__construct($model, $section);

        return new Response($this->generate());
    }

    // Override parent methods as needed
}
```

The same works for content elements by extending a legacy `Content*` class with
`#[AsContentElement(...)]`. While supported, creating elements/modules the legacy
way is discouraged for new code.

## Maker bundle

`contao/maker-bundle` (install with `--save-dev`) scaffolds the class, palette,
and template via `bin/console make:contao:content-element` and
`make:contao:frontend-module`.
