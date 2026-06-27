# Contao hook examples

Distilled from the Contao developer docs (getting-started/hooks, framework/hooks,
reference/hooks). All examples register listeners with `#[AsHook]` and invokable
services unless noted.

## `parseTemplate` — enrich a template (no return value)

Fires before a template is parsed. Signature: `__invoke(\Contao\Template $template): void`.

Add a variable to every `fe_page` template:

```php
namespace App\EventListener;

use Contao\CoreBundle\DependencyInjection\Attribute\AsHook;
use Contao\Template;

#[AsHook('parseTemplate')]
class ParseTemplateListener
{
    public function __invoke(Template $template): void
    {
        if ('fe_page' === $template->getName() || 0 === strpos($template->getName(), 'fe_page_')) {
            $template->foobar = 'foobar';
        }
    }
}
```

Expose a helper closure on front end templates (constructor injection of a
Symfony service):

```php
namespace App\EventListener;

use Contao\CoreBundle\DependencyInjection\Attribute\AsHook;
use Contao\CoreBundle\Security\ContaoCorePermissions;
use Contao\FrontendTemplate;
use Contao\Template;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;

#[AsHook('parseTemplate')]
class ParseTemplateListener
{
    public function __construct(
        private readonly AuthorizationCheckerInterface $authorizationChecker,
    ) {
    }

    public function __invoke(Template $template): void
    {
        if (!$template instanceof FrontendTemplate) {
            return;
        }

        $template->isMemberOf = fn ($groupId): bool =>
            $this->authorizationChecker->isGranted(ContaoCorePermissions::MEMBER_IN_GROUPS, $groupId);
    }
}
```

```html
<!-- templates/my_template.html5 -->
<?php if ($this->isMemberOf(1)): ?>
  <p>Member belongs to group ID 1!</p>
<?php endif; ?>
```

## `replaceInsertTags` — custom insert tag (must return string or false)

> Deprecated: stops working in Contao 6. Register custom insert tags with the
> dedicated insert-tag API (`#[AsInsertTag]`) instead. Kept here so you can
> recognise and migrate existing code.

Return a replacement string if you handle the tag; **return `false`** otherwise
so the next listener runs.

```php
namespace App\EventListener;

use Contao\CoreBundle\DependencyInjection\Attribute\AsHook;

#[AsHook('replaceInsertTags')]
class ReplaceInsertTagsListener
{
    public function __invoke(string $tag)
    {
        if ('mytag' !== $tag) {
            return false;
        }

        return 'mytag replacement';
    }
}
```

Parsing a parameterised tag like `{{mytag::foobar}}` — you split the parameters
yourself:

```php
public function __invoke(string $tag)
{
    [$name, $param] = explode('::', $tag) + [null, null];

    if ('mytag' !== $name) {
        return false;
    }

    return 'mytag replacement with parameter: '.$param;
}
```

The full signature also passes caching/flags context, which you rarely need:
`__invoke(string $tag, bool $useCache, string $cachedValue, array $flags, array $tags, array $cache, int $_rit, int $_cnt)`.

## `parseArticles` — enrich news entries before render

Signature: `(\Contao\FrontendTemplate $template, array $newsEntry, \Contao\Module $module): void`.

```php
namespace App\EventListener;

use Contao\CoreBundle\DependencyInjection\Attribute\AsHook;
use Contao\FrontendTemplate;
use Contao\Module;
use Contao\UserModel;

#[AsHook('parseArticles')]
class ParseArticlesListener
{
    public function __invoke(FrontendTemplate $template, array $newsEntry, Module $module): void
    {
        $author = UserModel::findById($newsEntry['author']);
        $template->set('author', $author->row());
    }
}
```

## Attribute on a method (one class, multiple hooks, with priority)

```php
class ParseArticlesListener
{
    #[AsHook('parseArticles', priority: 100)]
    public function onParseArticles(FrontendTemplate $template, array $newsEntry, Module $module): void
    {
        // …
    }
}
```

## Service-tag registration (alternative to the attribute)

```yaml
# config/services.yaml
services:
    App\EventListener\ActivateAccountListener:
        tags:
            - { name: contao.hook, hook: activateAccount, method: onAccountActivation, priority: 100 }
```

Tag options: `name` (always `contao.hook`), `hook` (the hook name), `method`
(optional; defaults to the inferred method or `__invoke`), `priority` (optional,
default `0`).

## Legacy `$GLOBALS['TL_HOOKS']` form (recognise, do not write)

Old extensions registered a `[class, method]` callback per hook and the core
iterated them manually. The modern `#[AsHook]` attribute replaces this entirely.

```php
// How the core historically invoked a hook (illustrative):
foreach ($GLOBALS['TL_HOOKS']['activateAccount'] as $callback) {
    $this->import($callback[0]);
    $this->{$callback[0]}->{$callback[1]}($objMember, $this);
}
```

Note that hooks requiring a return value pass it along the chain, e.g.
`compileFormFields` must return `$arrFields`.
