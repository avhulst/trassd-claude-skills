# Views, navigation, and tabs

Backing docs: `book/extend-admin.rst`, `cookbook/add-admin-tabs.rst`.

## Full add/edit form flow

A list links to add and edit views. Each form lives under a `ResourceTabs`
parent that loads the resource once and hosts form tabs as children.

```php
namespace App\Admin;

use App\Entity\Event;
use Sulu\Bundle\AdminBundle\Admin\Admin;
use Sulu\Bundle\AdminBundle\Admin\View\ToolbarAction;
use Sulu\Bundle\AdminBundle\Admin\View\ViewBuilderFactoryInterface;
use Sulu\Bundle\AdminBundle\Admin\View\ViewCollection;

class EventAdmin extends Admin
{
    const EVENT_FORM_KEY = 'event_details';
    const EVENT_LIST_VIEW = 'app.events_list';
    const EVENT_ADD_FORM_VIEW = 'app.event_add_form';
    const EVENT_EDIT_FORM_VIEW = 'app.event_edit_form';

    public function __construct(private ViewBuilderFactoryInterface $viewBuilderFactory) {}

    public function configureViews(ViewCollection $viewCollection): void
    {
        // List, wired to the add/edit form views and toolbar actions.
        $viewCollection->add(
            $this->viewBuilderFactory->createListViewBuilder(static::EVENT_LIST_VIEW, '/events')
                ->setResourceKey(Event::RESOURCE_KEY)
                ->setListKey('events')
                ->addListAdapters(['table'])
                ->setAddView(static::EVENT_ADD_FORM_VIEW)    // opened by sulu_admin.add
                ->setEditView(static::EVENT_EDIT_FORM_VIEW)  // opened on row click
                ->addToolbarActions([
                    new ToolbarAction('sulu_admin.add'),
                    new ToolbarAction('sulu_admin.delete'),
                ])
        );

        // Add: ResourceTabs parent + a form-tab child.
        $viewCollection->add(
            $this->viewBuilderFactory->createResourceTabViewBuilder(static::EVENT_ADD_FORM_VIEW, '/events/add')
                ->setResourceKey(Event::RESOURCE_KEY)
                ->setBackView(static::EVENT_LIST_VIEW)
        );
        $viewCollection->add(
            $this->viewBuilderFactory->createFormViewBuilder(static::EVENT_ADD_FORM_VIEW . '.details', '/details')
                ->setResourceKey(Event::RESOURCE_KEY)
                ->setFormKey(static::EVENT_FORM_KEY)
                ->setTabTitle('sulu_admin.details')
                ->setEditView(static::EVENT_EDIT_FORM_VIEW)  // where to go after create
                ->addToolbarActions([
                    new ToolbarAction('sulu_admin.save'),
                    new ToolbarAction('sulu_admin.delete'),
                ])
                ->setParent(static::EVENT_ADD_FORM_VIEW)
        );

        // Edit: ResourceTabs parent (note :id) + a form-tab child.
        $viewCollection->add(
            $this->viewBuilderFactory->createResourceTabViewBuilder(static::EVENT_EDIT_FORM_VIEW, '/events/:id')
                ->setResourceKey(Event::RESOURCE_KEY)
                ->setBackView(static::EVENT_LIST_VIEW)
        );
        $viewCollection->add(
            $this->viewBuilderFactory->createFormViewBuilder(static::EVENT_EDIT_FORM_VIEW . '.details', '/details')
                ->setResourceKey(Event::RESOURCE_KEY)
                ->setFormKey(static::EVENT_FORM_KEY)
                ->setTabTitle('sulu_admin.details')
                ->addToolbarActions([
                    new ToolbarAction('sulu_admin.save'),
                    new ToolbarAction('sulu_admin.delete'),
                ])
                ->setParent(static::EVENT_EDIT_FORM_VIEW)
        );
    }
}
```

Key points:

- `setAddView` / `setEditView` on the list connect it to the form views; edit
  appends the clicked row's ID.
- `ResourceTabs` views hold the loaded resource so switching tabs does not
  refetch; children read it without reloading.
- Path parameters are colon-prefixed (`:id`); child paths are appended to the
  parent path, so a form child only declares its own segment (`/details`).
- `setBackView` adds a back button; the add form's `setEditView` defines where
  to navigate after a successful create.
- `resourceKey` selects the data; `formKey` selects the field set — same
  resource + different `formKey` spreads a resource across multiple tabs.
- Toolbar actions come from the `formToolbarActionRegistry` /
  `listToolbarActionRegistry`. `ToolbarAction(key, optionsArray)`; subclasses
  like `DropdownToolbarAction` exist. Add them via `addToolbarActions`.

## Adding a tab to an existing resource

Attach a new form view as a child of the target resource's edit-form view. Here
a "Socials" tab is added to pages, gated by a webspace permission check.

```php
use Sulu\Bundle\AdminBundle\Admin\Admin;
use Sulu\Bundle\AdminBundle\Admin\View\ToolbarAction;
use Sulu\Bundle\AdminBundle\Admin\View\ViewBuilderFactoryInterface;
use Sulu\Bundle\AdminBundle\Admin\View\ViewCollection;

class SocialAdmin extends Admin
{
    public function __construct(
        private ViewBuilderFactoryInterface $viewBuilderFactory,
        private WebspaceManagerInterface $webspaceManager,
        private SecurityCheckerInterface $securityChecker
    ) {}

    public function configureViews(ViewCollection $viewCollection): void
    {
        if (!$this->hasSomeWebspacePermission()) {
            return;
        }

        $viewCollection->add(
            $this->viewBuilderFactory
                ->createPreviewFormViewBuilder('sulu_page.page_edit_form.socials', '/socials')
                ->disablePreviewWebspaceChooser()
                ->setResourceKey('pages')
                ->setFormKey('page_socials')
                ->setTabTitle('Socials')
                ->setTabPriority(256)
                ->addToolbarActions([new ToolbarAction('sulu_admin.save_with_publishing')])
                ->addRouterAttributesToFormRequest(['parentId', 'webspace'])
                ->setPreviewCondition('nodeType == 1')
                ->setTitleVisible(true)
                ->setTabOrder(1536)
                ->setParent(PageAdmin::EDIT_FORM_VIEW)
        );
    }

    private function hasSomeWebspacePermission(): bool
    {
        foreach ($this->webspaceManager->getWebspaceCollection()->getWebspaces() as $webspace) {
            if ($this->securityChecker->hasPermission(
                PageAdmin::SECURITY_CONTEXT_PREFIX . $webspace->getKey(),
                PermissionTypes::EDIT
            )) {
                return true;
            }
        }

        return false;
    }
}
```

Notes:

- `setParent(PageAdmin::EDIT_FORM_VIEW)` makes the view appear as a new tab in
  the page edit form. `setTabOrder` / `setTabPriority` control placement.
- The form XML (`config/forms/page_socials.xml`) supplies the fields. Use
  slash-separated property names to nest values:
  `ext/social/twitter_title`, `ext/social/twitter_image`, etc.
- Showing the tab does not persist its data. For values outside template data
  you extend the entity and typically add content merger/mapper/normalizer
  pieces — that storage layer is beyond this skill.

## Registering the Admin class as a service (bundles)

App projects get autoconfiguration. In a bundle, register explicitly and tag:

```yaml
app.social_admin:
    class: App\Admin\SocialAdmin
    arguments:
        - '@Sulu\Bundle\AdminBundle\Admin\View\ViewBuilderFactoryInterface'
        - '@sulu_core.webspace.webspace_manager'
        - '@sulu_security.security_checker'
    tags:
        - { name: 'sulu.admin' }
        - { name: 'sulu.context', context: 'admin' }
```

Confirm with `php bin/console debug:container --tag=sulu.admin`.
