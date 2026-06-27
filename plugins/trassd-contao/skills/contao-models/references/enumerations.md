# Enumerations (Contao 5.3+)

Models can resolve a stored column value into a PHP enum.

## Declare the enum in the DCA

Add an `'enum'` key to the field naming the enum class:

```php
// contao/dca/tl_member.php
$GLOBALS['TL_DCA']['tl_member']['fields']['salutation'] = [
    'inputType' => 'select',
    'enum' => App\Data\Salutation::class,
];
```

## Resolve with getEnum()

The database stores only the enum's backing `value`. `Model::getEnum()`
resolves it back to an enum instance, or `null` if it cannot be resolved.

```php
$member = MemberModel::findById(42);

$member->salutation;            // raw stored value, e.g. 'ms'
$member->getEnum('salutation'); // App\Data\Salutation or null
```

## Type-safe access with a fallback

Wrap `getEnum()` in a dedicated accessor that declares the enum return type and
supplies a fallback case when the stored value does not resolve:

```php
use App\Data\Salutation;
use Contao\MemberModel;

class SalutableMember extends MemberModel
{
    public function getSalutation(): Salutation
    {
        return $this->getEnum('salutation') ?? Salutation::mx;
    }
}
```
