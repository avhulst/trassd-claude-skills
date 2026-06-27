---
name: contao-dca-config
description: >-
  Configure Data Container Arrays (DCA) — tables, palettes, fields, callbacks,
  and PaletteManipulator. Triggers when adding or editing files under
  contao/dca/*.php or extending an existing DCA (e.g. adding a field to
  tl_news, defining a new tl_* table, or registering a DCA callback).
---

# Contao DCA configuration

Data Container Arrays describe a *Data Container*: how Contao lists records,
renders back end edit forms, and saves each record. They are the central
configuration mechanism for any editable data in Contao.

## Where a DCA lives

- One file per table under `contao/dca/`, named exactly like the table:
  table `tl_news` → `contao/dca/tl_news.php`.
- All table names start with the prefix `tl_`.
- Configuration goes into the `$GLOBALS['TL_DCA'][<table>]` array.
- DCA files of all active bundles are loaded in order, so a later file can
  **override** anything an earlier one defined — this is how you extend core
  tables like `tl_news` or `tl_content`.
- After changing a DCA, rebuild the Symfony cache for the production
  environment (dev picks up changes immediately). Use `debug:dca` to inspect
  the merged DCA of a table (e.g. to find which palette to extend).

## Top-level structure

```php
// contao/dca/tl_example.php
$GLOBALS['TL_DCA']['tl_example'] = [
    'config'      => [ /* table behaviour, driver, sql keys, callbacks */ ],
    'list'        => [ /* sorting, label, operations */ ],
    'palettes'    => [ /* back end form layout */ ],
    'subpalettes' => [ /* fields shown when a selector is active */ ],
    'fields'      => [ /* column definitions */ ],
];
```

### config

Defines the table's behaviour. Common keys:

```php
use Contao\DC_Table;

'config' => [
    'dataContainer'    => DC_Table::class, // driver: DC_Table, DC_File or DC_Folder
    'ptable'           => 'tl_vendor',     // parent table (child records)
    'ctable'           => ['tl_parts'],    // child tables
    'enableVersioning' => true,
    'switchToEdit'     => true,
    'sql'              => ['keys' => ['id' => 'primary', 'tstamp' => 'index']],
],
```

The available drivers are `DC_Table` (database records, the common case),
`DC_File` and `DC_Folder`. Custom drivers must extend `\Contao\DataContainer`.

### fields

The `fields` key defines the table columns. From those settings Contao decides
which widget to render, access control, and sort/filter eligibility. A field
combines an `inputType` (the widget), an `eval` array (widget configuration),
and an `sql` definition (the database column):

```php
$GLOBALS['TL_DCA']['tl_example']['fields']['location'] = [
    'label'     => &$GLOBALS['TL_LANG']['tl_example']['location'],
    'exclude'   => true,
    'search'    => true,
    'inputType' => 'text',
    'eval'      => ['tl_class' => 'w50', 'maxlength' => 255, 'mandatory' => true],
    'sql'       => ['type' => 'string', 'length' => 255, 'default' => ''],
];
```

- A table managed by `DC_Table` always needs `id` and `tstamp` fields.
- `sql` accepts a Doctrine schema array (`['type' => 'string', 'length' => 255,
  'default' => '']`) or the legacy SQL string (`"varchar(255) NOT NULL default ''"`).
  Since Contao 5.7 a field **without** `sql` becomes a *virtual field* stored in
  a JSON column instead of its own column.
- `label` is optional; without it Contao falls back to
  `$GLOBALS['TL_LANG'][<table>][<field>]`.

Common `inputType` widgets: `text`, `textarea`, `select`, `checkbox`, `radio`,
`fileTree`, `pageTree`, `picker`, `password`. Common `eval` keys: `mandatory`,
`maxlength`, `tl_class`, `rgxp` (validation), `multiple`, `includeBlankOption`,
`submitOnChange`, `rte`. See [references/fields.md](references/fields.md) for
the field-key, `eval`, `rgxp`, relation, and SQL reference tables.

### palettes & subpalettes

A palette is a **string** listing the fields of the back end edit form. Fields
are separated by commas; a semicolon starts a new collapsible fieldset whose
`{name_legend}` placeholder is translated via `TL_LANG`:

```php
'palettes' => [
    'default' => '{title_legend},title,alias;{meta_legend:collapsed},description',
],
```

A `__selector__` field can toggle subpalettes or switch between multiple main
palettes. Subpalette fields appear when the selector is active:

```php
'palettes' => [
    '__selector__' => ['addImage'],
    'default'      => '{title_legend},title,addImage',
],
'subpalettes' => [
    'addImage' => 'singleSRC,size',   // shown when the addImage checkbox is on
],
```

Enable `submitOnChange` on the selector field so subpalettes appear/disappear
immediately. Field width is controlled per field via the `eval` `tl_class`
(`w50`, `clr`, `long`, …). See [references/palettes.md](references/palettes.md)
for select-based subpalettes, multiple main palettes, and `tl_class` values.

## Extending palettes safely — PaletteManipulator

Never hand-edit a palette string with `.=` or `str_replace()`; it is fragile.
Use `Contao\CoreBundle\DataContainer\PaletteManipulator` to add/remove fields
and legends at defined positions, then apply the change to the live DCA:

```php
use Contao\CoreBundle\DataContainer\PaletteManipulator;

PaletteManipulator::create()
    ->addField('location', 'title_legend', PaletteManipulator::POSITION_APPEND)
    ->applyToPalette('default', 'tl_news')
    ->applyToPalette('internal', 'tl_news');
```

Key methods: `addField()`, `removeField()`, `addLegend()`,
`applyToPalette($palette, $table)`, `applyToSubpalette($subpalette, $table)`.
The position is the third argument; the four constants are `POSITION_BEFORE`,
`POSITION_AFTER` (relative to a field), `POSITION_PREPEND`, `POSITION_APPEND`
(relative to a legend). Adding a field requires both registering it under
`fields` *and* applying it to a palette — otherwise it stays hidden in the
back end. See [references/palette-manipulator.md](references/palette-manipulator.md).

## Callbacks

Callbacks are entry points for custom code bound to a specific table; they let
you modify the DCA or the record at runtime. Specify legacy callbacks as
`['Class', 'method']` arrays (or anonymous functions) directly in the DCA, e.g.
`config.onload`, `list.sorting.child_record`, `fields.<field>.options`,
`fields.<field>.save`.

```php
'config' => [
    'onload_callback' => [['App\\Dca\\TlExample', 'onLoad']],
],
'fields' => [
    'country' => [
        'inputType'       => 'select',
        'options_callback' => ['App\\Dca\\TlExample', 'getCountries'],
        // ...
    ],
],
```

The **modern, preferred** approach is an autowired service tagged with the
`#[AsCallback]` attribute (`Contao\CoreBundle\DependencyInjection\Attribute\AsCallback`),
which keeps logic out of the DCA file:

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

`AsCallback` takes `table`, `target`, optional `method` (defaults to `__invoke`
on invokable services), and optional `priority`. The same registration is also
possible via the `contao.callback` service tag in YAML, or PHP annotations for
PHP 7 setups. See [references/callbacks.md](references/callbacks.md) for the
full list of callback targets, their parameters, and return expectations.

## Defining a new manageable table — checklist

1. Create `contao/dca/tl_<name>.php` and set
   `$GLOBALS['TL_DCA']['tl_<name>']`.
2. `config`: `dataContainer => DC_Table::class`, `sql.keys`, plus `ptable`/
   `ctable` for parent/child relationships.
3. `list`: `sorting` (mode + fields + flag), `label`, and `operations`
   (edit/delete/show).
4. `fields`: include `id` and `tstamp`, then your columns with
   `inputType` + `eval` + `sql`.
5. `palettes`: at least a `default` palette grouping the fields.
6. Register a back end module in `contao/config/config.php` via
   `$GLOBALS['BE_MOD'][<group>][<module>] = ['tables' => [...]];`.
7. Add labels/legends under `contao/languages/<lang>/tl_<name>.php`.

See [references/examples.md](references/examples.md) for fully worked
parent/child table definitions.
