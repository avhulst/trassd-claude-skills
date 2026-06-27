# Nesting, the re-render algorithm, and testing

## Each component is an isolated universe

By default, parent and child components are fully independent:

- A parent re-render does **not** re-render its children.
- A child model change does **not** change a matching model on the parent.
- An action triggered in a child runs only on the child, even if a same-named action exists on the
  parent (same for `data-model`, with special handling — see below).

You opt into cross-component behavior explicitly.

## Re-rendering a child when a parent prop changes: `updateFromParent`

```php
#[LiveProp(updateFromParent: true)]
public int $count = 0;
```

When the parent re-renders and the passed-in `count` value differs, the child makes a second AJAX
request to re-render itself. The prop name passed when rendering must match the property name with
`updateFromParent` (e.g. `{{ component('TodoFooter', { count: todos|length }) }}`); setting it via
`mount()` under a different name won't work.

A re-rendered child **keeps** its own writable LiveProp values — only `updateFromParent` props are
refreshed. To fully reset a child (including writable props), give it an `id`/`key` that changes:

```twig
{{ component('TodoFooter', { count: todos|length, id: 'todo-footer-'~todos|length }) }}
```

## Updating a parent model from a child: `dataModel`

Pass `dataModel` to bind a child model to a parent prop. When the child model changes, the parent
prop changes (and the parent re-renders):

```twig
{{ component('TextareaField', { dataModel: 'content' }) }}                 {# parent 'content' <-> child 'value' #}
{{ component('TextareaField', { dataModel: 'content:value' }) }}           {# explicit parentProp:childProp #}
{{ component('TextareaField', { dataModel: 'user.firstName:first user.lastName:last' }) }}  {# multiple, space-separated #}
```

Note: changing a child LiveProp on the **server** (during re-render or an action) does not propagate
to a parent sharing that model — only frontend model changes do.

The two ways to communicate child → parent: **emit events** (most flexible — see
`actions-and-events.md`) or **`dataModel`** (simple model synchronization).

## Lists: the `key` prop

When rendering a list of elements or child components, give each a unique `key` so the morphing
engine tracks identity across re-renders (otherwise the wrong items may be removed/reused):

```twig
{% for lineItem in lineItems %}
    <div key="{{ lineItem.id }}">{{ lineItem.name }}</div>   {# plain element #}
{% endfor %}

{% for lineItem in invoice.lineItems %}
    {{ component('InvoiceLineItemForm', { lineItem: lineItem, key: lineItem.id }) }}  {# child component #}
{% endfor %}
```

`key` generates the child's `id` attribute used for identity.

### The "new item" trap

A child rendered with a static key like `key: 'new_line_item'` won't refresh after you add a real
item, because its props didn't change so the engine reuses it (form fields keep stale data). Fix by
making the key dynamic (`key: 'new_line_item_'~lineItems|length`) or by resetting the component's
state inside its save action (re-instantiate the entity, `clearValidation()` if validatable).

## Smart re-render / morphing

New HTML is "morphed" onto the existing DOM (idiomorph), and changes made by external JavaScript are
mostly preserved:

- JS attribute change on a child element → **preserved**.
- JS-added element → **preserved**.
- JS-removed element that the component originally rendered → **lost** (re-added next render).
- JS-changed text → **lost** (restored from server).
- Element moved within the component → **lost** (re-added in original spot).

Controls:

- `data-live-ignore` — never update this element on re-render (rarely needed). Force it to re-render
  by changing its parent's `id`.
- `data-skip-morph` — overwrite an element's inner HTML instead of morphing it (attributes are still
  morphed). Useful for tricky elements like `<select>`.
- `id` (and `key`, which generates an `id`) — the identifier the morphing library uses to connect
  old and new HTML; changing it forces a complete re-render of that element/component.

## Passing content (blocks)

Blocks work as in Twig Components, with one caveat: on re-render, variables defined only in the
**outer** template are unavailable. Define variables **inside** the block if they must survive
re-rendering.

## Testing

Use the `InteractsWithLiveComponents` trait in a `KernelTestCase`:

```php
use Symfony\UX\LiveComponent\Test\InteractsWithLiveComponents;

$c = $this->createLiveComponent(name: 'MyComponent', data: ['foo' => 'bar']);
$c->render();                                   // assert on the HTML
$c->call('increase', ['amount' => 2]);          // call a LiveAction (files: ... for uploads)
$c->emit('increaseEvent', ['amount' => 2]);     // emit an event
$c->set('count', 99);                           // set a prop
$c->submitForm(['my_form' => ['input' => 'x']], 'save');
$response = $c->call('redirect')->response();   // inspect a redirect response
$c->actingAs($user);                            // authenticate
$component = $c->component();                    // the live component object in current state
```

Assertion helpers include `assertComponentEmitEvent(...)->withData(...)`,
`assertComponentNotEmitEvent(...)`, and `assertComponentDispatchBrowserEvent(...)->withPayload(...)`.
For `LiveCollectionType`, call `addCollectionItem` (via `LiveCollectionTrait`) once per row before
`submitForm`.

Use the Twig Component debug command to list/inspect components.
