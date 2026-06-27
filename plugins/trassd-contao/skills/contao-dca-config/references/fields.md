# DCA fields reference

The `fields` key defines table columns. Each field combines top-level keys, an
`eval` array, and an `sql` definition.

## Common top-level field keys

| Key | Description |
| --- | --- |
| `label` | Field label, usually `&$GLOBALS['TL_LANG'][<table>][<field>]`. Array of `[label, description]`. Optional — Contao falls back to the `TL_LANG` entry. |
| `default` | Default value for new records. |
| `exclude` | If true, the field is hidden from non-admins (grantable in user-group settings). |
| `search` | Include the field in the search panel and back end search. |
| `sorting` / `filter` | Include the field in the sorting / filter panel. |
| `flag` | Sorting/grouping mode (1 = initial letter asc, 2 = desc, 5–10 = day/month/year, 11/12 = asc/desc, …). |
| `inputType` | The back end widget to render (see below). |
| `options` | Static option list for `select`/`radio` (indexed or associative). |
| `options_callback` | Callback returning the option array (`['Class','Method']`). |
| `reference` | Labels for option values, usually a `TL_LANG` reference. |
| `foreignKey` | `table.field` — pull options from another table (id => field). |
| `eval` | Widget configuration array (see below). |
| `relation` | Relation to another table (see below). |
| `sql` | Database column definition (see below). Omit for a virtual field (Contao 5.7+). |
| `save_callback` / `load_callback` | Callbacks run when the field is saved / loaded. |

## Common `inputType` widgets

`text`, `textarea`, `select`, `checkbox`, `checkboxWizard`, `radio`,
`radioTable`, `password`, `textStore`, `fileTree`, `pageTree`, `picker`,
`inputUnit`, `imageSize`, `keyValueWizard`, `listWizard`, `tableWizard`,
`metaWizard`, `serpPreview`.

## Common `eval` keys

| Key | Description |
| --- | --- |
| `mandatory` | Field cannot be empty. |
| `maxlength` / `minlength` | Character bounds. |
| `maxval` / `minval` | Numeric bounds. |
| `tl_class` | Layout CSS class(es) — see palettes reference. |
| `rgxp` | Validation rule (see below). |
| `multiple` | Allow multiple values (text, select, radio, checkbox). |
| `size` | Number of fields / select size. |
| `includeBlankOption` | Add a blank option to a drop-down. |
| `blankOptionLabel` | Label for the blank option (default `-`). |
| `submitOnChange` | Submit the form when the value changes (needed for selectors). |
| `chosen` | Enhance a select with Chosen. |
| `rte` | Rich text editor (`tinyMCE`, `ace`, `ace|html`, …). |
| `datepicker` / `colorpicker` | Add the respective picker. |
| `isAssociative` | Treat ambiguous numeric option arrays as associative. |
| `fieldType` | `checkbox` or `radio` for file/page trees. |
| `filesOnly` / `files` / `extensions` / `path` | File tree configuration. |
| `dcaPicker` | Enable the general-purpose picker (bool or insert-tag config array). |
| `encrypt` | Store the value encrypted. |
| `doNotCopy` / `doNotSaveEmpty` / `doNotShow` | Behaviour toggles. |
| `helpwizard` + `explanation` | Show a help-wizard icon with explanatory text (`TL_LANG['XPL']`). |
| `targetColumn` | JSON storage column for a virtual field (default `jsonData`). |

## Regular expressions (`rgxp`)

`digit`, `natural`, `alpha`, `alnum`, `extnd`, `date`, `time`, `datim`,
`friendly`, `email`, `emails`, `url`, `httpurl`, `alias`, `folderalias`,
`phone`, `prcnt`, `locale`, `language`, `fieldname`, `custom` (with
`customRgxp` + `errorMsg`).

## SQL column definition

Use a Doctrine schema array (preferred) or a legacy SQL string:

| Doctrine array | SQL equivalent |
| --- | --- |
| `['type' => 'string', 'length' => 32, 'default' => '']` | `VARCHAR(32) NOT NULL DEFAULT ''` |
| `['type' => 'string', 'length' => 1, 'fixed' => true, 'default' => '']` | `CHAR(1) NOT NULL DEFAULT ''` |
| `['type' => 'integer', 'notnull' => false, 'unsigned' => true]` | `INT UNSIGNED NULL` |
| `['type' => 'binary', 'length' => 16, 'fixed' => true, 'notnull' => false]` | `BINARY(16) NULL` |

For the primary key column use
`['type' => 'integer', 'unsigned' => true, 'autoincrement' => true]` and declare
`'id' => 'primary'` under `config.sql.keys`.

## Relations

Set `foreignKey` and a `relation` array so models can resolve related records
via `\Contao\Model::getRelated()`:

```php
'pid' => [
    'foreignKey' => 'tl_vendor.name',
    'sql'        => ['type' => 'integer', 'unsigned' => true, 'default' => 0],
    'relation'   => ['type' => 'belongsTo', 'load' => 'lazy'],
],
```

`type`: `hasOne`, `hasMany`, `belongsTo`, `belongsToMany`. `load`: `lazy`
(default) or `eager`.

## Virtual fields (Contao 5.7+)

A field with no `sql` key is auto-mapped into a JSON column (`jsonData`) instead
of its own column, and is still readable via the model (`$model->field`).
Override the storage column with the `targetColumn` key. Auto-mapping is skipped
for Doctrine-entity DCAs, `notEditable` tables, and fields with an
`input_field_callback` or `save_callback`.
