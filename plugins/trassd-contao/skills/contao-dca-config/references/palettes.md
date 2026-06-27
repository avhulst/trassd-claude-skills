# DCA palettes reference

A palette is a string listing the fields of the back end edit form. Commas
separate field names; a semicolon starts a new collapsible fieldset, whose
`{name_legend}` placeholder is translated via `TL_LANG`. Append `:collapsed` to
a legend to render the fieldset collapsed by default.

```php
'{title_legend},headline,alias,author;{date_legend},date,time;{teaser_legend:collapsed},subheadline,teaser'
```

## Main palettes

Stored under the `palettes` key; at minimum a `default` palette:

```php
'palettes' => [
    'default' => '{title_legend},title,alias,addImage',
],
```

## Subpalettes (checkbox selector)

A field listed in `__selector__` toggles a subpalette. The subpalette's fields
are appended after the selector field when it is active:

```php
'palettes' => [
    '__selector__' => ['addImage'],
    'default'      => '{title_legend},title,alias,addImage',
],
'subpalettes' => [
    'addImage' => 'singleSRC,size',
],
```

Enable `submitOnChange` in the selector field's `eval` so the subpalette
appears/disappears immediately.

## Subpalettes (select selector)

With a select as selector, the subpalette key is `fieldName_fieldValue`:

```php
'palettes' => [
    '__selector__' => ['selectField'],
    'default'      => '{title_legend},title,selectField',
],
'subpalettes' => [
    'selectField_value1' => 'field1,field2',
    'selectField_value2' => 'field3,field4',
],
```

## Multiple main palettes

A `__selector__` can switch the whole main palette; the palette key is the
selector's value:

```php
'palettes' => [
    '__selector__' => ['type'],
    'default' => '{title_legend},type',
    'text'    => '{title_legend},type,textField',
    'image'   => '{title_legend},type,imageField',
],
```

## Legend translations

```php
// contao/languages/en/tl_news.php
$GLOBALS['TL_LANG']['tl_news']['title_legend'] = 'Title and author';
$GLOBALS['TL_LANG']['tl_news']['date_legend']  = 'Date and time';
```

## Field layout (`tl_class` in `eval`)

The back end uses a two-column grid. Apply classes via the field's
`'eval' => ['tl_class' => '...']`:

| `tl_class` | Effect |
| --- | --- |
| `w25` / `w33` / `w50` / `w66` / `w75` | Float the field at the given width. |
| `clr` | Clear floats — required on a full-width field following a `w50` field. |
| `long` | Full available width. |
| `wizard` | Shorten the input to leave room for a wizard button. |
| `cbx` / `m12` / `cbx m12` | Extra height/padding for single checkboxes. |

Because floated fields can break the layout, always add `clr` to a full-width
field that immediately follows a `w50` field.
