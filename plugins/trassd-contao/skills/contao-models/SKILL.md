---
name: contao-models
description: Work with Contao models (Active Record over DCA tables) — defining a Model, finder methods, collections, customization, and enum casts. Triggers when adding a model under contao/models (or src/Model), querying records via a Model class (e.g. PageModel::findBy...), iterating a Model\Collection, or resolving a DCA column into a PHP enum.
---

# Contao Models

A Contao model is an Active Record object over a single DCA database table —
comparable to a Doctrine entity, but with static finder methods and a 1:1
mapping to a `tl_*` table. Every table has a corresponding model class
(`tl_page` → `PageModel`, `tl_news` → `NewsModel`, `tl_member` → `MemberModel`).

All model classes extend `Contao\Model`. Collections of models are
`Contao\Model\Collection`.

## The model class

A model maps to its table via the static `$strTable` property. Naming
convention: drop the `tl_` prefix, convert snake_case to PascalCase, append
`Model`.

```php
namespace App\Model;

use Contao\Model;

/**
 * Declare columns as @property for IDE support.
 *
 * @property string $hash
 */
class ExampleModel extends Model
{
    protected static $strTable = 'tl_example';
}
```

Defining your own model takes three steps: create the DCA, create the model
class, and register it. See [references/customization.md](references/customization.md).

## Finder methods

Every model exposes static finders. If no row matches, the return value is
`null` (never an empty collection).

```php
use Contao\PageModel;

$page  = PageModel::findByPk(2);                      // by primary key → Model
$page  = PageModel::findById(2);                      // alias of findByPk → Model
$page  = PageModel::findByIdOrAlias('index');         // ID or alias → Model
$page  = PageModel::findOneBy('adminEmail', $email);  // first match → Model
$pages = PageModel::findBy('language', 'de');         // all matches → Collection
$pages = PageModel::findAll();                        // every row → Collection
$total = PageModel::countBy('language', 'de');        // int
```

`findOneBy()` returns a `Model`; `findBy()` returns a `Collection`.
`findByPk()`, `findById()` and `findByIdOrAlias()` always return a `Model`.

### Late static binding

`Model` implements `__callStatic()`, so you can fold the column name into the
method and drop the first argument. Supported for `findOneBy()`, `findBy()` and
`countBy()`:

```php
$page  = PageModel::findOneByAdminEmail($email);
$pages = PageModel::findByLanguage('de');
$total = PageModel::countByLanguage('de');
```

### Reading and saving

```php
$id = $page->id;          // read a column
$page->alias = 'index';   // set a column
$page->save();            // persist
```

### Query options

Pass an options array as the last argument (after the value, or after the value
for late-static-binding calls). It is merged into the SQL query.

| Option   | Effect                                  | SQL        | Example                 |
|----------|-----------------------------------------|------------|-------------------------|
| `limit`  | Limit total records                     | `LIMIT`    | `5`                     |
| `offset` | Skip the first n records                | `OFFSET`   | `10`                    |
| `order`  | Sort by a field                         | `ORDER BY` | `'id DESC'`             |
| `return` | Force return type                       | —          | `'Model'`/`'Collection'`|
| `eager`  | Load related `hasOne`/`belongsTo` rows  | `LEFT JOIN`| `true`                  |
| `having` | Filter on aggregated/joined columns     | `HAVING`   | `'id = 1'`              |

```php
$pages = PageModel::findBy('pid', 1, ['limit' => 5, 'offset' => 10]);
$pages = PageModel::findByPid(1, ['order' => 'sorting']);
```

With `'eager' => true`, related records (for DCA relations of type `hasOne` /
`belongsTo`) load in the same query via a join; joined columns are prefixed
with `<column>__`, which you can then filter with `having`. Access a related
record with `$model->getRelated('fieldName')`. See
[references/relations-and-options.md](references/relations-and-options.md).

## Collections

A `Collection` wraps one or more models and is what multi-row finders return.
Iterate it with `foreach`; each item is a model instance.

```php
$pages = PageModel::findAll();

foreach ($pages as $page) {
    // $page is a PageModel
}

$titles = $pages->fetchEach('title');  // one column from each row
$rows   = $pages->fetchAll();          // all columns from each row
```

For complex conditions, pass arrays of clauses and bound values to `findBy()`,
and build a collection from specific IDs with `findMultipleByIds()`. See
[references/collections.md](references/collections.md).

## Enumerations (Contao 5.3+)

A DCA field can declare an `'enum'` class. Only the enum's backing `value` is
stored in the database; the model resolves it back to the PHP enum.

```php
// DCA: $GLOBALS['TL_DCA']['tl_member']['fields']['salutation'] = [
//   'inputType' => 'select',
//   'enum' => App\Data\Salutation::class,
// ];

$member = MemberModel::findById(42);
$member->salutation;             // raw string value, e.g. 'ms'
$member->getEnum('salutation');  // App\Data\Salutation or null
```

For type-safe access with a fallback, wrap `getEnum()` in a dedicated accessor:

```php
public function getSalutation(): Salutation
{
    return $this->getEnum('salutation') ?? Salutation::mx;
}
```

See [references/enumerations.md](references/enumerations.md).
