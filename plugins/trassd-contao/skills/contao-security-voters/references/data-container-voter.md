# DataContainer voter — full example

Restricting CRUD on a single DCA table with `AbstractDataContainerVoter`,
including the parent record permission and the list-view filtering needed to
avoid permission-denied exceptions.

## 1. Parent table DCA

```php
// contao/dca/tl_example_archive.php
use Contao\DC_Table;

$GLOBALS['TL_DCA']['tl_example_archive'] = [
    'config' => [
        'dataContainer' => DC_Table::class,
        'ctable' => ['tl_example_item'],
    ],
    // ...
];
```

## 2. Register two custom permissions

One controls *which* parent records the user may access, the other *which* CRUD
operations are granted on the table.

```php
// contao/config/config.php
$GLOBALS['TL_PERMISSIONS'][] = 'examples';  // accessible archive IDs
$GLOBALS['TL_PERMISSIONS'][] = 'examplep';  // allowed operations
```

## 3. Expose the permissions in the user-group DCA

(Can also be added to `tl_user`.)

```php
// contao/dca/tl_user_group.php
use Contao\CoreBundle\DataContainer\PaletteManipulator;

PaletteManipulator::create()
    ->addLegend('example_legend', 'amg_legend', PaletteManipulator::POSITION_AFTER)
    ->addField('examples', 'example_legend', PaletteManipulator::POSITION_APPEND)
    ->addField('examplep', 'example_legend', PaletteManipulator::POSITION_APPEND)
    ->applyToPalette('default', 'tl_user_group')
;

$GLOBALS['TL_DCA']['tl_user_group']['fields']['examples'] = [
    'inputType'  => 'checkbox',
    'foreignKey' => 'tl_example_archive.title',
    'eval'       => ['multiple' => true],
    'sql'        => "blob NULL",
];

$GLOBALS['TL_DCA']['tl_user_group']['fields']['examplep'] = [
    'inputType' => 'checkbox',
    'options'   => ['create', 'delete'],
    'reference' => &$GLOBALS['TL_LANG']['MSC'],
    'eval'      => ['multiple' => true],
    'sql'       => "blob NULL",
];
```

## 4. The voter

`AbstractDataContainerVoter` handles `supports*()` and abstaining for any other
table; you only declare the table and decide each action. Here we delegate the
fine-grained checks to the `AccessDecisionManager` so the registered
`contao_user.*` permissions are evaluated by the core voter.

```php
// src/Security/Voter/DataContainer/ExampleAccessVoter.php
namespace App\Security\Voter\DataContainer;

use Contao\CoreBundle\Security\ContaoCorePermissions;
use Contao\CoreBundle\Security\DataContainer\CreateAction;
use Contao\CoreBundle\Security\DataContainer\DeleteAction;
use Contao\CoreBundle\Security\DataContainer\ReadAction;
use Contao\CoreBundle\Security\DataContainer\UpdateAction;
use Contao\CoreBundle\Security\Voter\DataContainer\AbstractDataContainerVoter;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\AccessDecisionManagerInterface;

class ExampleAccessVoter extends AbstractDataContainerVoter
{
    public function __construct(
        private readonly AccessDecisionManagerInterface $accessDecisionManager,
    ) {
    }

    protected function getTable(): string
    {
        return 'tl_example_archive';
    }

    protected function hasAccess(
        TokenInterface $token,
        UpdateAction|CreateAction|ReadAction|DeleteAction $action,
    ): bool {
        // Must have access to the back end module at all
        if (!$this->accessDecisionManager->decide($token, [ContaoCorePermissions::USER_CAN_ACCESS_MODULE], 'example')) {
            return false;
        }

        return match (true) {
            $action instanceof CreateAction =>
                $this->accessDecisionManager->decide($token, ['contao_user.examplep.create']),
            $action instanceof ReadAction,
            $action instanceof UpdateAction =>
                $this->accessDecisionManager->decide($token, ['contao_user.examples'], $action->getCurrentId()),
            $action instanceof DeleteAction =>
                $this->accessDecisionManager->decide($token, ['contao_user.examples'], $action->getCurrentId())
                && $this->accessDecisionManager->decide($token, ['contao_user.examplep.delete']),
        };
        // Add further custom conditions here if needed.
    }
}
```

## 5. Filter the list view

The list view must not show records the user cannot read; otherwise loading
such a record triggers a permission-denied exception. Set the allowed root IDs
for non-admins in a `config.onload` listener:

```php
// src/EventListener/DataContainer/ExampleArchiveOnLoadListener.php
namespace App\EventListener\DataContainer;

use Contao\BackendUser;
use Contao\CoreBundle\DependencyInjection\Attribute\AsCallback;
use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;

#[AsCallback(table: 'tl_example_archive', target: 'config.onload')]
class ExampleArchiveOnLoadListener
{
    public function __construct(private readonly TokenStorageInterface $tokenStorage)
    {
    }

    public function __invoke(): void
    {
        $user = $this->tokenStorage->getToken()?->getUser();

        if (!$user instanceof BackendUser || $user->isAdmin) {
            return; // admins see everything
        }

        $root = (!$user->examples || !\is_array($user->examples)) ? [0] : $user->examples;

        $GLOBALS['TL_DCA']['tl_example_archive']['list']['sorting']['root'] = $root;
    }
}
```

When new parent records are created you typically need to grant the creating
user access to the new ID — mirror how the Contao core does it for news
archives (`tl_news_archive`).
