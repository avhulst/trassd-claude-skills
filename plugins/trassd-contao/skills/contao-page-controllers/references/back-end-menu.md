# Adding a back end menu entry for a back end route

A custom back end controller is reachable by URL but does not appear in the
menu automatically. Add a menu node by listening for the back end menu build
event (`contao.backend_menu_build`, constant
`ContaoCoreEvents::BACKEND_MENU_BUILD`).

Register with the `#[AsEventListener]` attribute and a **low priority** so the
listener runs after the Contao core listeners — otherwise the parent node
(e.g. `content`) you attach to does not exist yet.

```php
// src/EventListener/BackendMenuListener.php
namespace App\EventListener;

use App\Controller\BackendController;
use Contao\CoreBundle\Event\ContaoCoreEvents;
use Contao\CoreBundle\Event\MenuEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpFoundation\RequestStack;

#[AsEventListener(ContaoCoreEvents::BACKEND_MENU_BUILD, priority: -255)]
class BackendMenuListener
{
    public function __construct(private readonly RequestStack $requestStack)
    {
    }

    public function __invoke(MenuEvent $event): void
    {
        $factory = $event->getFactory();
        $tree = $event->getTree();

        // Only act on the main menu tree.
        if ('mainMenu' !== $tree->getName()) {
            return;
        }

        $contentNode = $tree->getChild('content');
        $request = $this->requestStack->getCurrentRequest();

        $node = $factory
            ->createItem('my-module', ['route' => BackendController::class])
            ->setLabel('My Modules')
            ->setLinkAttribute('title', 'Title')
            ->setLinkAttribute('class', 'my-module')
            // Symfony exposes the matched controller under the _controller attribute.
            ->setCurrent($request?->attributes->get('_controller') === BackendController::class)
        ;

        $contentNode->addChild($node);
    }
}
```

Using the route name `BackendController::class` matches the `name: self::class`
set on the controller's `#[Route]`. The node then highlights itself as current
by comparing the request's `_controller` attribute.
