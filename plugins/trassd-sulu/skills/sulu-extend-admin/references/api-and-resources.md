# REST API, resources, and metadata

Backing docs: `book/extend-admin.rst`.

## Entity with a resource key

The `RESOURCE_KEY` constant uniquely identifies the entity and is reused as the
list key and across resource/field-type config.

```php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Event
{
    const RESOURCE_KEY = 'events';

    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private $id;

    #[ORM\Column(type: 'string')]
    private $name;

    #[ORM\Column(type: 'datetime_immutable')]
    private $startDate;

    #[ORM\Column(type: 'datetime_immutable')]
    private $endDate;
}
```

## REST controller using the ListBuilder

The admin list needs pagination/search/sort. Use the `ListBuilder` abstraction
so responses match the conventions the list view expects. `cgetAction` loads
field descriptors from the list XML, builds an efficient Doctrine query, and
wraps the result in a `PaginatedRepresentation`.

```php
namespace App\Controller\Admin;

use App\Entity\Event;
use Sulu\Component\Rest\ListBuilder\Doctrine\DoctrineListBuilderFactoryInterface;
use Sulu\Component\Rest\ListBuilder\Metadata\FieldDescriptorFactoryInterface;
use Sulu\Component\Rest\ListBuilder\PaginatedRepresentation;
use Sulu\Component\Rest\RestHelperInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Serializer\SerializerInterface;

class EventController
{
    public function __construct(
        private FieldDescriptorFactoryInterface $fieldDescriptorFactory,
        private DoctrineListBuilderFactoryInterface $listBuilderFactory,
        private RestHelperInterface $restHelper,
        private SerializerInterface $serializer,
    ) {}

    public function cgetAction(): Response
    {
        $fieldDescriptors = $this->fieldDescriptorFactory->getFieldDescriptors(Event::RESOURCE_KEY);
        $listBuilder = $this->listBuilderFactory->create(Event::class);
        $this->restHelper->initializeListBuilder($listBuilder, $fieldDescriptors);

        $listRepresentation = new PaginatedRepresentation(
            $listBuilder->execute(),
            Event::RESOURCE_KEY,
            $listBuilder->getCurrentPage(),
            $listBuilder->getLimit(),
            $listBuilder->count()
        );

        return new JsonResponse($this->serializer->serialize($listRepresentation->toArray()));
    }

    // getAction / postAction / putAction / deleteAction return JSON serialized entities.
}
```

The `get`/`post`/`put` actions exchange JSON in the same shape, e.g.:

```json
{ "id": 1, "name": "Sulu Con 2020", "startDate": "2020-10-24T08:00:00", "endDate": "2020-10-25T18:00:00" }
```

## Routes

```yaml
# config/routes_admin.yaml
app.get_events:
    path: /admin/api/events
    controller: App\Controller\Admin\EventController::cgetAction
    methods: [GET]

app.get_event:
    path: /admin/api/events/{id}
    controller: App\Controller\Admin\EventController::getAction
    methods: [GET]

app.post_event:
    path: /admin/api/events
    controller: App\Controller\Admin\EventController::postAction
    methods: [POST]

app.put_event:
    path: /admin/api/events/{id}
    controller: App\Controller\Admin\EventController::putAction
    methods: [PUT]

app.delete_event:
    path: /admin/api/events/{id}
    controller: App\Controller\Admin\EventController::deleteAction
    methods: [DELETE]
```

Verify routing with `bin/adminconsole debug:router`. To expose actions to the
frontend, use `FOSJsRoutingBundle` (`options: ['expose' => true]`); Sulu also
auto-exposes names matching `(.+\.)?c?get_.*`. Check with
`bin/console fos:js-routing:debug`.

## Resource registration

Map the collection/detail route names to the resource key. The detail route
must be the one including the ID.

```yaml
# config/packages/sulu_admin.yaml
sulu_admin:
    resources:
        events:
            routes:
                list: app.get_events
                detail: app.get_event
```

## List metadata

`config/lists/events.xml`:

```xml
<?xml version="1.0" ?>
<list xmlns="http://schemas.sulu.io/list-builder/list">
    <key>events</key>
    <properties>
        <property name="id" visibility="no" translation="sulu_admin.id">
            <field-name>id</field-name>
            <entity-name>App\Entity\Event</entity-name>
        </property>
        <property name="name" visibility="always" searchability="yes" translation="sulu_admin.name">
            <field-name>name</field-name>
            <entity-name>App\Entity\Event</entity-name>
        </property>
        <property name="startDate" visibility="yes" translation="app.start_date" type="datetime">
            <field-name>startDate</field-name>
            <entity-name>App\Entity\Event</entity-name>
        </property>
    </properties>
</list>
```

- `visibility`: `yes`/`no` set the default but let the user toggle;
  `always`/`never` lock it.
- `searchability="yes"` includes the column in the list search field.
- `type` controls formatting (e.g. `datetime` localizes the value); extend types
  via the JS `listFieldTransformerRegistry`.
- `<field-name>` / `<entity-name>` let the `ListBuilder` join only what's needed.

## Form metadata

`config/forms/event_details.xml`:

```xml
<?xml version="1.0" ?>
<form xmlns="http://schemas.sulu.io/template/template"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://schemas.sulu.io/template/template http://schemas.sulu.io/template/form-1.0.xsd"
>
    <key>event_details</key>
    <properties>
        <property name="name" type="text_line" mandatory="true" colspan="12">
            <meta><title>sulu_admin.name</title></meta>
            <params><param name="headline" value="true"/></params>
        </property>
        <property name="startDate" type="date" mandatory="true" colspan="12">
            <meta><title>app.start_date</title></meta>
        </property>
    </properties>
</form>
```

- `type` is a field type from the JS `fieldRegistry`.
- `colspan` 1–12 sets width; `mandatory` makes the field required.
- `visibleCondition` / `disabledCondition` work like template property conditions.
- `<title>` uses the `admin` translation domain; add a `lang` attribute to use a
  literal string instead of a translation key.
- The JSON from the server must use keys matching each field `name`; the value
  format depends on the field `type`.

## Selection field types (relations, no JS)

Register reusable selection fields under `field_type_options` using the
`selection` / `single_selection` blueprints, then reference the new type from
the form XML.

```yaml
sulu_admin:
    field_type_options:
        selection:
            event_selection:
                default_type: 'list_overlay'
                resource_key: 'events'
                view:
                    name: 'app.event_edit_form'
                    result_to_view: { id: 'id' }
                types:
                    list_overlay:
                        adapter: 'table'
                        list_key: 'events'
                        display_properties: ['name']
                        icon: 'su-calendar'
                        label: 'app.events'
                        overlay_title: 'app.events'
        single_selection:
            single_event_selection:
                default_type: 'list_overlay'
                resource_key: 'events'
                types:
                    auto_complete:
                        display_property: 'name'
                        search_properties: ['name']
```

```xml
<property name="similar_events" type="event_selection">
    <meta><title>app.similar_events</title></meta>
</property>
<property name="parent_event" type="single_event_selection">
    <meta><title>app.parent_event</title></meta>
    <params><param name="type" value="auto_complete" /></params>
</property>
```

- `selection` types: auto-complete, full `List`, or `list_overlay`.
- `single_selection` adds a `single_select` dropdown type.
- `default_type` picks the type when the form XML does not override it via a param.
- The form shows the field, but the REST API must accept/persist these values too.
