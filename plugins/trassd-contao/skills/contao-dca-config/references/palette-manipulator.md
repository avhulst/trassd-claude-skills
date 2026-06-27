# PaletteManipulator reference

`Contao\CoreBundle\DataContainer\PaletteManipulator` edits palette strings
safely instead of using string concatenation or `str_replace()`. Build up the
changes with a fluent API, then apply them to a palette or subpalette.

## Adding a field

```php
use Contao\CoreBundle\DataContainer\PaletteManipulator;

PaletteManipulator::create()
    ->addField('custom_field', 'username')   // after the "username" field (default)
    ->applyToPalette('admin', 'tl_user');     // register in $GLOBALS['TL_DCA']
```

Adding a field requires both defining it under `fields` *and* applying it to a
palette here — otherwise it stays hidden in the back end.

## Position constants (third argument)

| Constant | Behaviour |
| --- | --- |
| `PaletteManipulator::POSITION_BEFORE` | Before the named parent **field**. |
| `PaletteManipulator::POSITION_AFTER` | After the named parent **field** (default). |
| `PaletteManipulator::POSITION_PREPEND` | Before the named parent **legend**. |
| `PaletteManipulator::POSITION_APPEND` | After the named parent **legend**. |

```php
PaletteManipulator::create()
    ->addField('location', 'title_legend', PaletteManipulator::POSITION_APPEND)
    ->applyToPalette('default', 'tl_news')
    ->applyToPalette('internal', 'tl_news');
```

## Adding a legend

`addLegend($name, $parent, $position)` — fields can then target the new legend.
Without a parent/position the legend is appended at the end.

```php
PaletteManipulator::create()
    ->addLegend('custom_legend', 'date_legend', PaletteManipulator::POSITION_BEFORE)
    ->addField('custom_field', 'custom_legend', PaletteManipulator::POSITION_APPEND)
    ->applyToPalette('default', 'tl_news');
```

## Removing a field

```php
PaletteManipulator::create()
    ->removeField('custom_field', 'name_legend')
    ->applyToPalette('admin', 'tl_user');
```

## Subpalettes

```php
PaletteManipulator::create()
    ->addField('custom_field', 'singleSRC')
    ->removeField('floating')
    ->applyToSubpalette('addImage', 'tl_content');
```

## Note on reuse

Each `applyTo*()` call does **not** clear the fields registered on the instance,
so chained applies accumulate. Create a fresh `PaletteManipulator::create()`
instance when you want an independent set of changes.
