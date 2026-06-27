# Controller security (security-context level)

Implement `SecuredControllerInterface` so the `SuluSecurityListener` performs
the permission check automatically. `getSecurityContext()` names the context to
check; `getLocale()` returns the locale the permissions apply to (usually from
the request). The listener derives the required permission type from the action
and returns **403** when the current user lacks it.

```php
<?php

namespace Acme\Bundle\ExampleBundle\Controller;

use FOS\RestBundle\Routing\ClassResourceInterface;
use Sulu\Component\Security\SecuredControllerInterface;
use Symfony\Component\HttpFoundation\Request;

class ExampleController implements ClassResourceInterface, SecuredControllerInterface
{
    public function cgetAction()
    {
        // GET list action — VIEW is checked by the listener
    }

    public function postAction()
    {
        // POST action — ADD is checked by the listener
    }

    public function getLocale(Request $request)
    {
        return $request->get('locale');
    }

    public function getSecurityContext(): string
    {
        return 'sulu.acme.example';
    }
}
```

The context returned here must be one declared by an `Admin::getSecurityContexts()`.
