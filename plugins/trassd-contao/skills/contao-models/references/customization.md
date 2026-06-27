# Creating a custom model

A custom model is ready after three steps.

## 1. Create the DCA

Define the table in a DCA file. The table name is the model's backing table.

```php
// contao/dca/tl_example.php
$GLOBALS['TL_DCA']['tl_example'] = [
    // palettes, fields, sql, ...
];
```

## 2. Create the model class

Naming convention: drop `tl_`, convert snake_case to PascalCase, append
`Model`. Set `$strTable` to the DCA table. Declare columns as `@property` for
IDE support, and add reusable record logic as instance methods.

```php
// src/Model/ExampleModel.php
namespace App\Model;

use Contao\Model;

/**
 * @property string $hash
 */
class ExampleModel extends Model
{
    protected static $strTable = 'tl_example';

    public function setHash(): void
    {
        $this->hash = md5($this->id);
    }
}
```

## 3. Register the model

Map the table to the class via `TL_MODELS`, so Contao knows which class to
instantiate for rows of that table.

```php
// contao/config/config.php
use App\Model\ExampleModel;

$GLOBALS['TL_MODELS']['tl_example'] = ExampleModel::class;
```

Custom finder methods are added as static methods on the model class, typically
delegating to the inherited `findBy()` / `findOneBy()` with preset conditions.
