# LiveProps: stateful properties

A `#[LiveProp]` is a "stateful" property: it is dehydrated, sent to the frontend, and re-hydrated
on every AJAX render so the component remembers it. Properties **without** `#[LiveProp]` are not
preserved across re-renders (services injected in the constructor are the common exception — they
are re-autowired, so they don't need to be props).

Pass initial values when rendering:

```twig
{{ component('RandomNumber', { max: 500 }) }}
```

## Writability is a security boundary

By default a `#[LiveProp]` is **read-only**: the client cannot change it and re-render. Opt in
explicitly:

```php
#[LiveProp(writable: true)]
public int $max = 1000;
```

- Only mark a prop writable if the user is genuinely meant to control it.
- A writable prop is request input — **never trust it**. Validate it, and authorize any action it
  drives. The client can send any value the type permits.

## Supported data types

LiveProps must be values that can be serialized to JavaScript: scalars (int/float/string/bool/null),
arrays of scalars, enums, `DateTime` objects, Doctrine entities, DTOs, or arrays of DTOs. Anything
more complex needs explicit hydration (below).

## Entities and hydration

When sent to the frontend a LiveProp is **dehydrated**; on the next request it is **hydrated** back.

```php
#[LiveProp]
public Post $post;
```

- A persisted entity dehydrates to its `id` and is re-fetched from the database on hydration.
- An unpersisted entity dehydrates to an empty array and re-hydrates as a `new Post()`.
- Arrays of entities / `DateTime` need PHPDoc the component can read, e.g. `/** @var Product[] */`.
  Collection-type extraction requires `phpdocumentor/reflection-docblock`.

## Writable object properties / array keys

Marking an entity prop `writable: true` lets the client change it to **any** entity in the database
(a known footgun). Prefer a path allow-list so only specific properties or array keys are writable:

```php
#[LiveProp(writable: ['title', 'content'])]
public Post $post;

#[LiveProp(writable: ['allow_markdown'])]
public array $options = ['allow_markdown' => true, 'allow_html' => false];
```

Then bind nested paths in the template: `data-model="post.title"`, `data-model="options.allow_markdown"`.
All other properties/keys stay read-only.

- `writable: true` on an **array** allows any key to be changed/added/removed.
- To allow swapping the entity itself **and** editing properties, include the identity constant:
  `#[LiveProp(writable: [LiveProp::IDENTITY, 'title', 'content'])]`. Changing identity only works
  for objects that dehydrate to a scalar (e.g. an entity `id`).

## DateTime format

```php
#[LiveProp(writable: true, format: 'Y-m-d')]
public ?\DateTime $publishOn = null;
```

Bind to a field that uses the same format (`<input type="date" data-model="publishOn">`).

## DTOs and custom hydration

A plain DTO works as a LiveProp if its type is correct. Rules: all public properties (or
getter/setter "fake" properties) are read & written via PropertyAccess; a property settable but not
gettable (or vice versa) throws; the DTO must have **no constructor arguments**. Use PHPDoc
(`/** @var AddressDto[] */`) for collections.

When the plain approach doesn't fit, three escape hatches (mutually exclusive with each other):

- **Serializer**: `#[LiveProp(useSerializerForHydration: true)]` (optionally with
  `serializationContext`).
- **Custom methods**: `#[LiveProp(hydrateWith: 'hydrateX', dehydrateWith: 'dehydrateX')]` pointing
  to methods on the component that convert to/from an array.
- **Hydration extension**: implement `HydrationExtensionInterface` (`supports`, `hydrate`,
  `dehydrate`) for a type you (de)hydrate repeatedly; autoconfiguration tags it, otherwise tag with
  `live_component.hydration_extension`. (Doctrine entities use this mechanism internally.)

## Reacting to updates: `onUpdated`

Run code right after a specific prop changes. The previous value is passed as the argument:

```php
#[LiveProp(writable: true, onUpdated: 'onQueryUpdated')]
public string $query = '';

public function onQueryUpdated($previousValue): void { /* $this->query is already the new value */ }
```

For writable object paths you can target a single key:
`#[LiveProp(writable: ['title'], onUpdated: ['title' => 'onTitleUpdated'])]`.

## Dynamic options: `modifier`

Configure a LiveProp's options at runtime from another prop. The method receives the original
`LiveProp` and must return a modified clone (all `with*()` methods are immutable):

```php
#[LiveProp(writable: true, modifier: 'modifyAddedDate')]
public ?\DateTimeImmutable $addedDate = null;

#[LiveProp] public string $dateFormat = 'Y-m-d';

public function modifyAddedDate(LiveProp $prop): LiveProp
{
    return $prop->withFormat($this->dateFormat);
}
```

Caution: don't reference another modifier-driven prop inside a modifier — it may not be hydrated yet.

## Lifecycle hooks

Methods can be tagged to run at specific stages (all support a `priority` argument — higher runs
first):

- `#[PostHydrate]` — right after state is loaded from the client.
- `#[PreDehydrate]` — just before state is serialized back to the client.
- `#[PreReRender]` — before re-render (not on the initial render); good for adjusting state.

For initial data setup, use the Twig Component `mount()` method or a `#[PostMount]` hook.
