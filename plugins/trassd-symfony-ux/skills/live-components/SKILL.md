---
name: live-components
description: Build reactive Symfony UX Live Components with `#[AsLiveComponent]`, `#[LiveProp]`, `#[LiveAction]` and events. Use when a Twig Component needs server-side reactivity, data binding that re-renders without a full page reload, live form validation as the user types, polling, or deferred/lazy loading.
---

# Symfony UX Live Components

Live Components extend Twig Components (`#[AsTwigComponent]`) to re-render automatically on the
frontend as the user interacts with them, driven by small AJAX requests. A property marked
`#[LiveProp]` is "stateful" (re-sent on every render); `data-model` binds form fields to those
props; `#[LiveAction]` methods are real Symfony controllers triggered from the DOM.

If the component does not yet exist as a plain Twig Component, build that first — Live Components
are a strict superset.

## Minimal anatomy

```php
use Symfony\UX\LiveComponent\Attribute\AsLiveComponent;
use Symfony\UX\LiveComponent\Attribute\LiveProp;
use Symfony\UX\LiveComponent\DefaultActionTrait;

#[AsLiveComponent]
class ProductSearch
{
    use DefaultActionTrait;

    #[LiveProp(writable: true)]
    public string $query = '';

    public function __construct(private ProductRepository $repo) {}

    public function getProducts(): array
    {
        return $this->repo->search($this->query);
    }
}
```

```twig
{# templates/components/ProductSearch.html.twig #}
<div {{ attributes }}>            {# single root element carrying {{ attributes }} is REQUIRED #}
    <input type="search" data-model="query">
    <ul>
        {% for product in this.products %}<li>{{ product.name }}</li>{% endfor %}
    </ul>
</div>
```

Three non-negotiables for any live component:
1. `#[AsLiveComponent]` on the class **and** `use DefaultActionTrait;` (provides the default
   re-render action). Forgetting the trait is the most common mistake.
2. Exactly **one** root HTML element, with `{{ attributes }}` applied to it.
3. Every property that must survive a re-render is a `#[LiveProp]`. Services injected via the
   constructor do not need to be props (they are re-autowired each request).

## Rules of thumb

- **Writable is opt-in and a security boundary.** A `#[LiveProp]` is read-only by default. Only
  `#[LiveProp(writable: true)]` (or a list of writable paths) lets the client change it. Never mark
  a prop writable unless the user is meant to control it, and never trust its value server-side —
  treat it like any request input (validate, authorize). See `references/live-props.md`.
- **`data-model` triggers re-render.** Typing into a bound field debounces ~150ms then re-renders.
  Use modifiers to tune this: `debounce(N)`, `on(change)` (re-render on blur), `norender` (update
  the value but defer rendering). See `references/data-binding.md`.
- **Actions are controllers.** `#[LiveAction]` methods support autowiring, `#[IsGranted]`, and
  returning a `RedirectResponse`. Trigger them with `data-action="live#action"` +
  `data-live-action-param="actionName"`. CSRF is handled automatically — do not weaken CORS
  (`Access-Control-Allow-Origin: *` breaks the built-in protection). See `references/actions-and-events.md`.
- **Forms get live validation for free.** Use `ComponentWithFormTrait` to render a whole Symfony
  form; fields re-validate as the user moves through them. For forms without the Form component,
  use `ValidatableComponentTrait`. See `references/forms-and-validation.md`.
- **Components communicate via events, not direct calls.** A child action never reaches the parent.
  Emit (`emit`/`emitUp`/`emitSelf`) and listen with `#[LiveListener('eventName')]`. See
  `references/actions-and-events.md`.
- **Each component is an isolated universe.** Parent re-render does not re-render children unless
  the prop is `#[LiveProp(updateFromParent: true)]`; a child model change does not touch the parent
  unless wired with `dataModel`. For lists of child components, give each a unique `key`. See
  `references/nesting-and-rendering.md`.
- **Defer heavy work.** Use `loading="defer"` (render after page load) or `loading="lazy"` (render
  when scrolled into view) plus a `placeholder` macro. Poll with `data-poll`. See
  `references/loading-polling-url.md`.
- **Don't fight the morphing engine.** Re-renders morph new HTML onto the DOM. Use `key` on looped
  elements, `data-live-ignore` to protect an element, and `id` to force a full re-render. See
  `references/nesting-and-rendering.md`.

## Choosing the right tool

| Need | Use |
|------|-----|
| Bind an input to a prop and re-render | `data-model` (+ modifiers) — `references/data-binding.md` |
| Run server logic on click / submit | `#[LiveAction]` — `references/actions-and-events.md` |
| Pass arguments to an action | `#[LiveArg]` — `references/actions-and-events.md` |
| Full Symfony form with live validation | `ComponentWithFormTrait` — `references/forms-and-validation.md` |
| Validate props without a Form | `ValidatableComponentTrait` — `references/forms-and-validation.md` |
| Talk between components | events + `#[LiveListener]` — `references/actions-and-events.md` |
| Entity/DTO/DateTime as a prop | hydration options — `references/live-props.md` |
| Sync a prop to the URL | `#[LiveProp(url: true)]` — `references/loading-polling-url.md` |
| Defer / lazy-load / poll | `loading=`, `data-poll` — `references/loading-polling-url.md` |
| Nested components, lists, morphing quirks | `key`, `updateFromParent`, `dataModel` — `references/nesting-and-rendering.md` |

## Reference files

- `references/live-props.md` — `#[LiveProp]` options, writability as a security boundary,
  hydration of entities/DTOs/DateTime, exposing object properties & array keys, `onUpdated`,
  `modifier`, lifecycle hooks.
- `references/data-binding.md` — `data-model`, all modifiers, `name=""` binding, checkboxes/
  selects/radios, updating a model manually and from JavaScript.
- `references/actions-and-events.md` — `#[LiveAction]`, `#[LiveArg]`, CSRF, redirects, file
  uploads/downloads, emitting & listening (`#[LiveListener]`, scoping), browser events.
- `references/forms-and-validation.md` — `ComponentWithFormTrait`, submitting via a LiveAction,
  `formValues`, collection types, `ValidatableComponentTrait`, real-time validation.
- `references/loading-polling-url.md` — loading states (`data-loading`, `aria-busy`), deferred/
  lazy loading & placeholders, polling, URL binding.
- `references/nesting-and-rendering.md` — parent/child isolation, `updateFromParent`, `dataModel`,
  `key` for lists, the smart re-render/morphing algorithm, `data-live-ignore`, testing.
