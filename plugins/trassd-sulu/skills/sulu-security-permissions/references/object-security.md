# Per-object permissions

Protect individual objects (not just whole contexts). Three parts: the
permission tab in the form, the resource mapping, and the controller interface.

## 1. Add the permission tab in the Admin

```php
<?php

namespace Sulu\Bundle\ExampleBundle\Admin;

use Sulu\Bundle\AdminBundle\Admin\Admin;
use Sulu\Bundle\AdminBundle\Admin\View\ToolbarAction;
use Sulu\Bundle\AdminBundle\Admin\View\ViewBuilderFactoryInterface;
use Sulu\Bundle\AdminBundle\Admin\View\ViewCollection;

class ExampleAdmin extends Admin
{
    public function __construct(private ViewBuilderFactoryInterface $viewBuilderFactory)
    {
    }

    public function configureViews(ViewCollection $viewCollection): void
    {
        $viewCollection->add(
            $this->viewBuilderFactory
                ->createFormViewBuilder('sulu_example.edit_form.permissions', '/permissions')
                ->setResourceKey('permissions')
                ->setFormKey('permission_details')
                ->addRequestParameters(['resourceKey' => 'example'])
                ->setTabCondition('_permissions.security')
                ->setTabTitle('sulu_security.permissions')
                ->addToolbarActions([new ToolbarAction('sulu_admin.save')])
                ->setParent(static::EDIT_FORM_VIEW)
        );
    }
}
```

The `resourceKey` request parameter (`example`) ties the tab to a resource.

## 2. Map the resource to a context and class

```yaml
resources:
    example:
        routes:
            list: 'get_examples'
            detail: 'get_example'
        security_context: 'sulu_admin.example'
        security_class: 'App\\Entity\\Example'
```

## 3. Implement SecuredObjectControllerInterface

Combine it with `SecuredControllerInterface`. `getSecuredClass()` must return the
same identifier as `security_class`; `getSecuredObjectId()` extracts the id from
the request. For list actions, filter by permission with `setPermissionCheck()`.

```php
<?php

namespace Acme\Bundle\ExampleBundle\Controller;

use FOS\RestBundle\Routing\ClassResourceInterface;
use Sulu\Component\Security\Authorization\AccessControl\SecuredObjectControllerInterface;
use Sulu\Component\Security\Authorization\PermissionTypes;
use Sulu\Component\Security\SecuredControllerInterface;
use Symfony\Component\HttpFoundation\Request;

class ExampleController implements
    ClassResourceInterface,
    SecuredControllerInterface,
    SecuredObjectControllerInterface
{
    public function cgetAction()
    {
        // ... build the list ...
        $listBuilder->setPermissionCheck($this->getUser(), PermissionTypes::VIEW);
        $listResponse = $listBuilder->execute(); // only rows the user may VIEW
    }

    public function getLocale(Request $request)
    {
        return $request->get('locale');
    }

    public function getSecurityContext(): string
    {
        return 'sulu.acme.example';
    }

    public function getSecuredClass(): string
    {
        return Example::class;
    }

    public function getSecuredObjectId(Request $request)
    {
        return $request->get('id');
    }
}
```

The `SuluSecurityListener` performs the object-level check the same way it does
for context-level security.
