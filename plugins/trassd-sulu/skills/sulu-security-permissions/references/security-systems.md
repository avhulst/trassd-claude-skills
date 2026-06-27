# Security systems

A role belongs to a system; the firewall logs a user in only if it has a role in
the request's assigned system. Default system: `Sulu`.

## Webspace security system

Set the system in the webspace configuration:

```xml
<security>
    <system>Website</system>
</security>
```

Restrict pages, media and custom entities to logged-in users with a role by
enabling `permission-check`:

```xml
<security permission-check="true">
    <system>example</system>
</security>
```

Enable user-context caching to avoid leaking restricted content from the cache.

## Custom security system

For an intranet/extranet not tied to a webspace, declare the system in an
`Admin` class:

```php
<?php

namespace App\Sulu\Admin;

use Sulu\Bundle\AdminBundle\Admin\Admin;

class ExtranetAdmin extends Admin
{
    public const SYSTEM = 'Extranet';

    public function getSecurityContexts()
    {
        return [self::SYSTEM => []];
    }
}
```

Register a request listener that assigns the system to the request via the
`SystemStore` (service `sulu_security.system_store`,
`Sulu\Bundle\SecurityBundle\System\SystemStoreInterface`). It must run after
Sulu's `SystemListener` and before Symfony's `FirewallListener`:

```php
<?php

namespace App\Sulu\Security;

use App\Sulu\Admin\ExtranetAdmin;
use Sulu\Bundle\SecurityBundle\System\SystemStoreInterface;
use Symfony\Bundle\SecurityBundle\Security\FirewallMap;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;
use Symfony\Component\Security\Http\FirewallMapInterface;

class SecuritySystemSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private SystemStoreInterface $systemStore,
        private FirewallMapInterface $map,
    ) {
        if (!$map instanceof FirewallMap) {
            throw new \LogicException(\sprintf('Expected "%s" but got "%s".', FirewallMap::class, \get_class($map)));
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::REQUEST => [
                ['processSecuritySystem', 9],
            ],
        ];
    }

    public function processSecuritySystem(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $config = $this->map->getFirewallConfig($event->getRequest());
        if (!$config) {
            return;
        }

        if ('extranet' === $config->getName()) {
            $this->systemStore->setSystem(ExtranetAdmin::SYSTEM);
        }
    }
}
```
