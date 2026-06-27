---
name: contao-hooks
description: >-
  Register and use Contao hooks — the #[AsHook] attribute, hook listeners, and
  the available hook signatures. Triggers when adding a hook listener or
  reacting to a Contao lifecycle/event hook (e.g. parseTemplate, parseArticles,
  replaceInsertTags, updatePersonalData) in a Contao bundle or app.
---

# Contao hooks

Hooks are Contao's own extension points into the core (and some extension
bundles). At a fixed point in the core's execution flow, every callable
registered for that hook is called in turn. They predate and are **distinct
from Symfony events**: a hook is a Contao concept with a per-hook fixed
signature, not an `Event` object dispatched through the event dispatcher.

## When to use a hook vs. a Symfony event

- **Prefer modern APIs / Symfony events** for your own extension points and
  wherever Contao offers one. If you write event-driven logic in your own code,
  use the Symfony Event Dispatcher, not hooks.
- **Use a hook** only to plug into a core/bundle extension point that is exposed
  *as a hook* — there is no other way in.
- Some hooks are themselves deprecated in favour of a dedicated API. Notably
  `replaceInsertTags` is deprecated and stops working in Contao 6; register
  custom insert tags via the dedicated insert-tag API (the `#[AsInsertTag]`
  attribute) instead. See `references/hook-examples.md`.

## Registering a listener (modern)

Tag a service with the `#[AsHook]` attribute from
`Contao\CoreBundle\DependencyInjection\Attribute\AsHook`. With autoconfigured
services under `App\` / `src/`, one file is all you need.

```php
namespace App\EventListener;

use Contao\CoreBundle\DependencyInjection\Attribute\AsHook;
use Contao\Template;

#[AsHook('parseTemplate')]
class ParseTemplateListener
{
    public function __invoke(Template $template): void
    {
        // …
    }
}
```

The attribute constructor is `AsHook(string $hook, ?string $method = null, ?int $priority = null)`.

### `__invoke` vs. a named method

- **Attribute on the class** → the `__invoke` method is called (invokable
  service). Cleanest for one hook per class.
- **Attribute on a method** → that method handles the hook, so one class can
  serve several hooks:

```php
#[AsHook('parseArticles', priority: 100)]
public function onParseArticles(FrontendTemplate $template, array $newsEntry, Module $module): void
{
    // …
}
```

### Priority

`priority` is optional (default `0`). Higher runs earlier.

- `priority` **> 0** → runs **before** legacy (`$GLOBALS['TL_HOOKS']`) hooks.
- `priority` **= 0** → runs in extension load order, interleaved with legacy hooks.
- `priority` **< 0** → runs **after** legacy hooks.

## Match the hook's signature exactly

Every hook has a **fixed signature** defined by the core — the exact parameters
passed in and whether a return value is expected. You must match it. Some hooks
expect no return (`parseTemplate`); others require you to return a value that is
passed along the chain (`replaceInsertTags`, `compileFormFields`, …). Look up
each hook's parameters and return contract in the
[Contao hooks reference](https://docs.contao.org/dev/reference/hooks/) before
implementing.

Two concrete signatures (full bodies in `references/hook-examples.md`):

- **`parseTemplate`** — `__invoke(Template $template): void`. Fires before a
  template is parsed; no return value. Use it to add/transform template
  variables or expose helper closures.
- **`replaceInsertTags`** — `__invoke(string $tag, …): string|false`. Fires for
  an unknown insert tag. Return the replacement **string** if you handle the
  tag; you **must** return `false` otherwise so the next listener gets a chance.
  (Deprecated — see above.)

## Legacy registration (recognise, don't write)

The old approach registers callbacks in the `$GLOBALS['TL_HOOKS']` array (a
`[class, method]` callback per hook), and the core iterates them manually. You
will see this in older code; the modern equivalent is `#[AsHook]`. You may also
register via the `contao.hook` service tag (`name`, `hook`, optional `method`,
optional `priority`) when attributes are not an option — see
`references/hook-examples.md`.

## References

- `references/hook-examples.md` — full listener examples (`parseTemplate`,
  `replaceInsertTags`, `parseArticles`, constructor injection), the service-tag
  and legacy `$GLOBALS['TL_HOOKS']` forms.
- [Hooks framework article](https://docs.contao.org/dev/framework/hooks/) ·
  [Hooks reference](https://docs.contao.org/dev/reference/hooks/) — authoritative
  list of every hook and its signature.
