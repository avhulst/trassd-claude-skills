# DCA callbacks reference

Callbacks are entry points for custom code bound to a specific DCA table. They
work like hooks but are table-scoped: register one or more callbacks for a
target and Contao runs them when that event fires. Depending on the target the
callback receives specific arguments and is expected to return specific data (or
`void`). Anonymous functions are also allowed.

## Registration

There are three ways to register, listed from most to least preferred:

### 1. `#[AsCallback]` attribute (preferred)

Tag an autowired service. Constructor arguments: `table`, `target`, optional
`method` (defaults to `__invoke` for invokable services), optional `priority`.

```php
namespace App\EventListener\DataContainer;

use Contao\CoreBundle\DependencyInjection\Attribute\AsCallback;
use Contao\DataContainer;

#[AsCallback(table: 'tl_module', target: 'list.label.group', priority: 100)]
class ModuleCallbackListener
{
    public function __invoke(string $group, string $mode, string $field, array $record, DataContainer $dc): string
    {
        return $group;
    }
}
```

### 2. `contao.callback` service tag (YAML)

```yaml
# config/services.yaml
services:
    App\EventListener\DataContainer\ModuleCallbackListener:
        tags:
            - { name: contao.callback, table: tl_module, target: list.label.group, method: onGroupCallback, priority: 100 }
```

Tag options: `name` (always `contao.callback`), `table`, `target`, optional
`method`, optional `priority` (default `0`, which runs after legacy callbacks).

### 3. Legacy DCA array / annotation

Directly in the DCA as `['Class', 'method']` arrays (or anonymous functions),
e.g. `$GLOBALS['TL_DCA']['tl_x']['config']['onload_callback'][] = [...]`. PHP
annotations (`@Callback(table=..., target=...)`) remain available for PHP 7.

## Callback targets

### Global (`config.*`)

`config.onload`, `config.oncreate`, `config.onbeforesubmit`, `config.onsubmit`,
`config.ondelete`, `config.oncut`, `config.oncopy`, `config.oncreate_version`,
`config.onrestore_version`, `config.onundo`, `config.oninvalidate_cache_tags`,
`config.onshow`, `config.onpalette`.

`config.onload` runs when the DataContainer initializes — the place to check
permissions or mutate the DCA at runtime (e.g. set a field's default or change
`mandatory`).

### Listing (`list.*`)

`list.sorting.paste_button`, `list.sorting.child_record`, `list.sorting.header`,
`list.sorting.panel_callback.subpanel`, `list.label.group`, `list.label.label`,
`list.global_operations.<OPERATION>.button`,
`list.operations.<OPERATION>.button`.

`list.sorting.child_record` is required when listing child records (parent
sorting mode) — without it no records render.

### Field (`fields.<FIELD>.*`)

`attributes`, `options`, `input_field`, `load`, `save`, `wizard`, `xlabel`,
`eval.url`, `eval.title_tag`.

`options` returns the option array for a select/radio; `save` validates or
transforms a value on save (throw an exception to show an error); `load`
transforms the value on load.

### Buttons

`edit.buttons`, `select.buttons`.

## Front end note

These callbacks normally run in the back end. Some also run from front end
modules (notably the member modules), in which case the arguments differ — there
is no `\Contao\DataContainer` instance in the front end. Guard for that case.
