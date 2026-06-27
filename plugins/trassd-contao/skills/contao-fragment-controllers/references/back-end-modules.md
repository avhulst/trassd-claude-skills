# Back end modules

A back end module is a back end **navigation entry**, registered the historical
way via `$GLOBALS['BE_MOD']`. The first level is a *category* (e.g. `content`,
`design`); the second level is the module key.

```php
// contao/config/config.php
$GLOBALS['BE_MOD']['content']['my_module'] = [
    'tables' => ['tl_my_module'],
];
```

Clicking the module loads the DCA of the relevant table; what happens next is
driven entirely by that DCA.

## Supported keys

| Key | Purpose |
| --- | --- |
| `tables` | Array of DCA tables the module manages (list child tables too). |
| `stylesheet` | Extra CSS loaded in the module context, e.g. `['bundles/mymodule/styles.css']`. |
| `javascript` | Extra JS loaded in the module context. |
| `callback` | A class with `public function generate(): string` rendering raw output. |
| `disablePermissionChecks` | `bool` — when `true`, the module is excluded from permission settings and checks are skipped. |
| `hideInNavigation` | `bool` — hide from the main navigation (still linkable). |
| `<custom-key>` | A `[class/serviceId, method]` callback invoked when `&key=<custom-key>` is in the query. |

```php
$GLOBALS['BE_MOD']['content']['my_module'] = [
    'tables'     => ['tl_my_module'],
    'javascript' => ['bundles/mymodule/scripts.js'],
    'stylesheet' => ['bundles/mymodule/styles.css'],
];
```

### Simple callback output

```php
$GLOBALS['BE_MOD']['content']['my_module'] = [
    'callback' => \App\Contao\BackendModule::class,
];
```

```php
namespace App\Contao;

class BackendModule
{
    public function generate(): string
    {
        return 'string content';
    }
}
```

This is the old, inflexible mechanism. For dependency injection prefer custom
back end routes instead.

### Custom action keys

A DCA operation can point at `key=<name>`, which Contao looks up in the module
definition and executes:

```php
// contao/dca/tl_theme.php
$GLOBALS['TL_DCA']['tl_theme']['list']['operations']['exportTheme'] = [
    'href' => 'key=exportTheme',
    // …
];
```
```php
// contao/config/config.php
$GLOBALS['BE_MOD']['design']['themes'] = [
    'exportTheme' => ['Contao\Theme', 'exportTheme'],
];
```

The first array element may be a class name **or** a service ID.

## Labels

Provide a `modules.xlf` at `contao/languages/<ISO>/modules.xlf` with
`MOD.<module>.0` (name) and `MOD.<module>.1` (description):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xliff version="1.1">
  <file datatype="php" source-language="en">
    <body>
      <trans-unit id="MOD.my_module.0"><source>My Module</source></trans-unit>
      <trans-unit id="MOD.my_module.1"><source>Manage entries of my module</source></trans-unit>
    </body>
  </file>
</xliff>
```

## Editing tl_content from your own back end module

To manage content elements (the `tl_content` table) under your own parent table:

1. **Declare the child table** in your DCA `config`:
   ```php
   $GLOBALS['TL_DCA']['tl_example']['config']['ctable'] = ['tl_content'];
   ```
2. **Add a list operation** that opens the content table:
   ```php
   $GLOBALS['TL_DCA']['tl_example']['list']['operations']['edit'] = [
       'href' => 'table=tl_content',
       'icon' => 'edit.svg',
   ];
   ```
3. **Allow `tl_content` in the back end module** so it is an accepted table:
   ```php
   $GLOBALS['BE_MOD']['content']['example'] = [
       'tables' => ['tl_example', 'tl_content'],
   ];
   ```
4. **Set the `ptable` dynamically** (the default is empty) via a
   `loadDataContainer` hook, only for back end requests in your module:
   ```php
   #[AsHook('loadDataContainer')]
   class SetPtableForContentListener
   {
       public function __construct(
           private RequestStack $requestStack,
           private ScopeMatcher $scopeMatcher,
       ) {}

       public function __invoke(string $table): void
       {
           if ('tl_content' !== $table) {
               return;
           }
           $request = $this->requestStack->getCurrentRequest();
           if (null === $request || !$this->scopeMatcher->isBackendRequest($request)) {
               return;
           }
           if ('example' === $request->query->get('do')) {
               $GLOBALS['TL_DCA']['tl_content']['config']['ptable'] = 'tl_example';
           }
       }
   }
   ```

Then render those elements in the front end as shown in
`references/templates-and-advanced.md`.
