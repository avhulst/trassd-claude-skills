# Worked DCA examples

Two complete tables for a vendor/parts admin module: `tl_vendor` (parent) and
`tl_parts` (child). Adapt the names and fields to your own data.

## Parent table: `tl_vendor`

```php
// contao/dca/tl_vendor.php
use Contao\DataContainer;
use Contao\DC_Table;

$GLOBALS['TL_DCA']['tl_vendor'] = [
    'config' => [
        'dataContainer'    => DC_Table::class,
        'ctable'           => ['tl_parts'],   // has child records
        'enableVersioning' => true,
        'switchToEdit'     => true,
        'sql'              => ['keys' => ['id' => 'primary', 'tstamp' => 'index']],
    ],
    'list' => [
        'sorting' => [
            'mode'        => DataContainer::MODE_SORTED,
            'fields'      => ['name'],
            'flag'        => DataContainer::SORT_INITIAL_LETTER_ASC,
            'panelLayout' => 'search,limit',
        ],
        'label'      => ['fields' => ['name'], 'format' => '%s'],
        'operations' => [
            'edit'       => ['href' => 'table=tl_parts', 'icon' => 'edit.svg'],
            'editheader' => ['href' => 'act=edit',       'icon' => 'header.svg'],
            'delete'     => ['href' => 'act=delete',     'icon' => 'delete.svg'],
            'show'       => ['href' => 'act=show',       'icon' => 'show.svg'],
        ],
    ],
    'palettes' => [
        'default' => '{vendor_legend},name;{address_legend},street,postal,city,country',
    ],
    'fields' => [
        'id'     => ['sql' => ['type' => 'integer', 'unsigned' => true, 'autoincrement' => true]],
        'tstamp' => ['sql' => ['type' => 'integer', 'unsigned' => true, 'default' => 0]],
        'name'   => [
            'search'    => true,
            'inputType' => 'text',
            'eval'      => ['tl_class' => 'w50', 'maxlength' => 255, 'mandatory' => true],
            'sql'       => ['type' => 'string', 'length' => 255, 'default' => ''],
        ],
        'street' => [
            'inputType' => 'text',
            'eval'      => ['tl_class' => 'w50', 'maxlength' => 255, 'mandatory' => true],
            'sql'       => ['type' => 'string', 'length' => 255, 'default' => ''],
        ],
        'country' => [
            'inputType'        => 'select',
            'options_callback' => static fn (): array =>
                \Contao\System::getContainer()->get('contao.intl.countries')->getCountries(),
            'eval'             => ['tl_class' => 'w50', 'mandatory' => true, 'includeBlankOption' => true],
            'sql'              => ['type' => 'string', 'length' => 2, 'default' => ''],
        ],
        // ... postal, city analogous to street
    ],
];
```

## Child table: `tl_parts`

```php
// contao/dca/tl_parts.php
use Contao\DataContainer;
use Contao\DC_Table;

$GLOBALS['TL_DCA']['tl_parts'] = [
    'config' => [
        'dataContainer'    => DC_Table::class,
        'enableVersioning' => true,
        'ptable'           => 'tl_vendor',   // belongs to a parent record
        'sql'              => ['keys' => ['id' => 'primary', 'tstamp' => 'index']],
    ],
    'list' => [
        'sorting' => [
            'mode'                 => DataContainer::MODE_PARENT,
            'fields'               => ['name'],
            'headerFields'         => ['name'],
            'panelLayout'          => 'search,limit',
            'child_record_callback' => static fn (array $row): string =>
                '<div class="tl_content_left">'.$row['name'].' ['.$row['number'].']</div>',
        ],
        'operations' => [
            'edit'   => ['href' => 'act=edit',   'icon' => 'edit.svg'],
            'delete' => ['href' => 'act=delete', 'icon' => 'delete.svg'],
            'show'   => ['href' => 'act=show',   'icon' => 'show.svg'],
        ],
    ],
    'palettes' => [
        'default' => '{parts_legend},name,number,description,singleSRC',
    ],
    'fields' => [
        'id'     => ['sql' => ['type' => 'integer', 'unsigned' => true, 'autoincrement' => true]],
        'pid'    => [
            'foreignKey' => 'tl_vendor.name',
            'sql'        => ['type' => 'integer', 'unsigned' => true, 'default' => 0],
            'relation'   => ['type' => 'belongsTo', 'load' => 'lazy'],
        ],
        'tstamp' => ['sql' => ['type' => 'integer', 'unsigned' => true, 'default' => 0]],
        'name'   => [
            'search'    => true,
            'flag'      => 1,   // group child records by initial letter
            'inputType' => 'text',
            'eval'      => ['tl_class' => 'w50', 'maxlength' => 255, 'mandatory' => true],
            'sql'       => ['type' => 'string', 'length' => 255, 'default' => ''],
        ],
        'description' => [
            'inputType' => 'textarea',
            'eval'      => ['tl_class' => 'clr', 'rte' => 'tinyMCE', 'mandatory' => true],
            'sql'       => ['type' => 'text', 'notnull' => false],
        ],
        'singleSRC' => [
            'inputType' => 'fileTree',
            'eval'      => [
                'tl_class'   => 'clr',
                'fieldType'  => 'radio',
                'filesOnly'  => true,
                'extensions' => \Contao\Config::get('validImageTypes'),
                'mandatory'  => true,
            ],
            'sql' => ['type' => 'binary', 'length' => 16, 'notnull' => false, 'fixed' => true],
        ],
        // ... number analogous to name
    ],
];
```

## Back end module

```php
// contao/config/config.php
$GLOBALS['BE_MOD']['content']['parts'] = [
    'tables' => ['tl_vendor', 'tl_parts'],
];
```

## Translations

```php
// contao/languages/en/modules.php
$GLOBALS['TL_LANG']['MOD']['parts'] = ['Parts', 'Manage vendors and parts.'];

// contao/languages/en/tl_vendor.php
$GLOBALS['TL_LANG']['tl_vendor']['vendor_legend']  = 'Vendor';
$GLOBALS['TL_LANG']['tl_vendor']['address_legend'] = 'Address';
$GLOBALS['TL_LANG']['tl_vendor']['name']           = ['Name', 'Name of the vendor.'];
```

## Extending an existing table

To add a field to a core table such as `tl_news`, create
`contao/dca/tl_news.php`, register the field, and apply it to the target
palette(s) with the PaletteManipulator:

```php
// contao/dca/tl_news.php
use Contao\CoreBundle\DataContainer\PaletteManipulator;

$GLOBALS['TL_DCA']['tl_news']['fields']['location'] = [
    'label'     => ['Location', 'Location of the news entry, if applicable.'],
    'inputType' => 'text',
    'eval'      => ['tl_class' => 'w50', 'maxlength' => 255],
    'sql'       => ['type' => 'string', 'length' => 255, 'default' => ''],
];

PaletteManipulator::create()
    ->addField('location', 'title_legend', PaletteManipulator::POSITION_APPEND)
    ->applyToPalette('default', 'tl_news')
    ->applyToPalette('internal', 'tl_news');
```

Use `debug:dca tl_news` to discover which palettes exist before extending them.
