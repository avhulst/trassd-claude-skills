# Registering custom back end access rights

To add a privilege that the Contao core `contao_user` voter understands (so you
can check it as `contao_user.<permission>`), you must register it and surface it
in the user/group DCAs.

## 1. Register the permission

```php
// contao/config/config.php
$GLOBALS['TL_PERMISSIONS'][] = 'my_permissions';
```

## 2. Add the field to `tl_user` and `tl_user_group`

The field must exist in **both** DCAs. Same definition for each — only the
target palettes differ.

```php
// contao/dca/tl_user.php  (and tl_user_group.php)
use Contao\CoreBundle\DataContainer\PaletteManipulator;

$GLOBALS['TL_DCA']['tl_user']['fields']['my_permissions'] = [
    'exclude'   => true,
    'inputType' => 'checkbox',
    'eval'      => ['multiple' => true],
    'options'   => [
        'first_permission'  => 'First permission',
        'second_permission' => 'Second permission',
    ],
    'sql' => ['type' => 'blob', 'notnull' => false],
];

PaletteManipulator::create()
    ->addLegend('my_legend', null)
    ->addField('my_permissions', 'my_legend', PaletteManipulator::POSITION_APPEND)
    ->applyToPalette('extend', 'tl_user')
    ->applyToPalette('custom', 'tl_user')
;
```

In `tl_user_group.php` apply to the `default` palette instead:

```php
PaletteManipulator::create()
    ->addLegend('my_legend', null)
    ->addField('my_permissions', 'my_legend', PaletteManipulator::POSITION_APPEND)
    ->applyToPalette('default', 'tl_user_group')
;
```

## 3. Check the permission

```php
// src/Controller/BackendController.php
namespace App\Controller;

use Contao\CoreBundle\Exception\AccessDeniedException;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;
use Twig\Environment;

#[Route('/contao/my-backend-route', name: BackendController::class, defaults: ['_scope' => 'backend'])]
class BackendController
{
    public function __construct(
        private readonly Environment $twig,
        private readonly AuthorizationCheckerInterface $auth,
    ) {
    }

    public function __invoke(): Response
    {
        // Admins should always pass; otherwise require the privilege
        if (!$this->auth->isGranted('ROLE_ADMIN')
            && !$this->auth->isGranted('contao_user.my_permissions', 'first_permission')) {
            throw new AccessDeniedException('Not enough permissions to access this controller.');
        }

        return new Response($this->twig->render('my_backend_route.html.twig', []));
    }
}
```

No explicit login check is needed: with the routing parameters above the Contao
firewall already enforces an authenticated back end user for this route.
