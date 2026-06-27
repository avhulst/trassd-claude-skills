# Actions, arguments, files, and events

## LiveActions

A live component needs one "default action" to re-render — that comes from `DefaultActionTrait`.
Custom server-side logic goes in methods tagged `#[LiveAction]`. After the method runs, the
component re-renders automatically.

```php
use Symfony\UX\LiveComponent\Attribute\LiveAction;

#[LiveAction]
public function resetMax(): void
{
    $this->max = 1000;
}
```

Trigger it from the DOM (the action name is passed as a Stimulus action parameter):

```twig
<button data-action="live#action" data-live-action-param="resetMax">Reset</button>

{# equivalent helper #}
<button {{ live_action('resetMax') }}>Reset</button>
```

Modifiers like `debounce(300)` work too: `data-live-action-param="debounce(300)|save"` or
`{{ live_action('save', {}, {'debounce': 300}) }}`.

**Live actions are real Symfony controllers.** You can:
- Autowire services into the method signature.
- Apply controller attributes (e.g. `#[IsGranted]`) to the class or a single action.
- Extend `AbstractController` to use shortcuts (`addFlash`, `redirectToRoute`, …).

```php
#[LiveAction]
public function resetMax(LoggerInterface $logger): void { /* ... */ }
```

## Arguments with `#[LiveArg]`

Pass arguments as Stimulus action params; receive them with `#[LiveArg]`. Use `#[LiveArg('frontName')]`
when the DOM param name differs from the PHP parameter name:

```twig
<button {{ live_action('addItem', {'id': item.id, 'itemName': 'Custom'}) }}>Add</button>
```

```php
#[LiveAction]
public function addItem(#[LiveArg] int $id, #[LiveArg('itemName')] string $name): void { /* ... */ }
```

## CSRF protection

Actions are POSTed with a custom `Accept` header that is set and validated automatically — you get
CSRF protection for free via browser same-origin/CORS policies. **Do not** set permissive CORS like
`Access-Control-Allow-Origin: *`, which defeats it. (CSRF protection is disabled in test mode.)

## Redirecting

Return a `RedirectResponse` from an action (extend `AbstractController` for the shortcut):

```php
#[LiveAction]
public function save()
{
    $this->addFlash('success', 'Saved!');
    return $this->redirectToRoute('app_random_number');
}
```

## File uploads

Files are not sent by default. Use the `files` modifier on the action to send pending files; the
component receives them in the normal `$request->files` bag.

```twig
<input type="file" name="my_file" />
<button data-action="live#action" data-live-action-param="files(my_file)|myAction">Upload</button>
{# files (no arg) sends all pending files; chain files(a)|files(b) for several #}
```

```php
#[LiveAction]
public function myAction(Request $request): void
{
    $file = $request->files->get('my_file');
    // multiple files: name="multiple[]" + $request->files->all('multiple')
}
```

## File downloads

Live actions cannot return file responses directly. Redirect to a route that serves the file, and
add `data-turbo="false"` to the root element so Turbo doesn't prefetch (and double-trigger) the URL.

## Events: communicating between components

Events are how any two components on the page talk; a child action never directly reaches the
parent. Three ways to emit:

```twig
<button data-action="live#emit" data-live-event-param="productAdded">Add</button>
```

```php
use Symfony\UX\LiveComponent\ComponentToolsTrait;
// in a class using ComponentToolsTrait:
$this->emit('productAdded');
```

```javascript
this.component.emit('productAdded');
```

Listen with `#[LiveListener]` (listeners are themselves LiveActions, so they autowire and trigger a
re-render):

```php
use Symfony\UX\LiveComponent\Attribute\LiveListener;

#[LiveListener('productAdded')]
public function incrementProductCount(): void { $this->productCount++; }
```

### Passing data to listeners

Emit with scalar extra data, receive it with `#[LiveArg]` (entity type-hints work, just like an
action):

```php
$this->emit('productAdded', ['product' => $product->getId()]);

#[LiveListener('productAdded')]
public function onAdded(#[LiveArg] Product $product): void { /* ... */ }
```

In Twig add `data-live-<name>-param="..."` alongside `data-live-event-param`.

### Scoping emits

By default an event reaches **all** components on the page. Narrow it:

- To parents only: `live#emitUp` / `$this->emitUp('...')`.
- To a specific component name: `name(ProductList)|productAdded` /
  `$this->emit('productAdded', componentName: 'ProductList')`.
- To self only: `live#emitSelf` / `$this->emitSelf('...')`.

## Browser/JavaScript events

Dispatch a DOM event on the component's root element (useful to signal e.g. a modal to close):

```php
$this->dispatchBrowserEvent('modal:close');
$this->dispatchBrowserEvent('product:created', ['product' => $product->getId()]); // becomes event.detail
```

Listen with a normal Stimulus controller / `window.addEventListener('product:created', e => e.detail.product)`.
