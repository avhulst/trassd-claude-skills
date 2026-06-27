# Doctrine broadcasting & async Mercure streams

## Mark the entity

```php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\UX\Turbo\Attribute\Broadcast;

#[ORM\Entity]
#[Broadcast]
class Book
{
    #[ORM\Column, ORM\Id, ORM\GeneratedValue(strategy: 'AUTO')]
    public ?int $id = null;

    #[ORM\Column]
    public string $title = '';
}
```

Every persisted create/update/remove of a `#[Broadcast]` entity renders the
broadcast template and pushes the resulting Turbo Stream to all connected
clients.

### Repeating the attribute / options

`#[Broadcast]` is repeatable, so the same change can render several templates
for different topics (e.g. a list view and a detail view):

```php
#[Broadcast(topics: ['@="book_detail" ~ entity.getId()', 'books'], template: 'book_detail.stream.html.twig', private: true)]
#[Broadcast(topics: ['@="book_list" ~ entity.getId()', 'books'], template: 'book_list.stream.html.twig', private: true)]
class Book { /* ... */ }
```

Options: `transports` (string[]), `topics` (string[], default derived from
FQCN + id), `template`. Mercure also supports `private`, `sse_id`,
`sse_type`, `sse_retry`. Topics beginning with `@=` are evaluated as
Expression Language.

## Subscribe in the page

```html+twig
{# preferred (UX 3.1+) #}
<twig:Turbo:Stream:From topics="App\\Entity\\Book" />
<twig:Turbo:Stream:From topics="App\\Entity\\Book" private transport="hub2" />

{# or the function form #}
{{ turbo_stream_from('App\\Entity\\Book') }}

{# subscribe to a single entity instance #}
<div id="book_{{ book.id }}" {{ turbo_stream_listen(book) }}></div>

{# all instances of a class, on a specific transport #}
<div id="books" {{ turbo_stream_listen('App\\Entity\\Book', 'hub2') }}></div>
```

`<twig:Turbo:Stream:From>` renders a `<turbo-mercure-stream-source>` element.

## Broadcast template (required blocks)

UX Turbo looks for `templates/broadcast/{ClassName}.stream.html.twig` and it
must define `create`, `update`, and `remove` (each may be empty). Variables:
`entity`, `id`, `action`, `options`.

```html+twig
{# templates/broadcast/Book.stream.html.twig #}
{% block create %}
    <turbo-stream action="append" targets="#books">
        <template>
            <div id="{{ 'book_' ~ id }}">{{ entity.title }} (#{{ id }})</div>
        </template>
    </turbo-stream>
{% endblock %}

{% block update %}
    <turbo-stream action="update" targets="#book_{{ id }}">
        <template>{{ entity.title }} (#{{ id }}, updated)</template>
    </turbo-stream>
{% endblock %}

{% block remove %}
    <turbo-stream action="remove" targets="#book_{{ id }}"></turbo-stream>
{% endblock %}
```

A block may contain multiple actions (e.g. update both a list and a detail
region); actions targeting missing DOM elements are ignored.

Configure namespace → template-prefix mapping in `config/packages/turbo.yaml`
under `turbo.broadcast.entity_template_prefixes` (default
`App\Entity\: broadcast/`).

## Async chat without Doctrine (manual publish)

For non-entity updates, publish directly to a Mercure hub and subscribe with
`turbo_stream_listen('topic')` / `<twig:Turbo:Stream:From topics="topic" />`:

```php
use Symfony\Component\Mercure\HubInterface;
use Symfony\Component\Mercure\Update;

$hub->publish(new Update(
    'chat',
    $this->renderView('chat/message.stream.html.twig', ['message' => $data['message']])
));
```

```html+twig
{# chat/message.stream.html.twig #}
<turbo-stream action="append" targets="#messages">
    <template><div>{{ message }}</div></template>
</turbo-stream>
```

With multiple hubs configured, inject the specific `HubInterface` service
(e.g. `$hub2`) to choose the transport.
