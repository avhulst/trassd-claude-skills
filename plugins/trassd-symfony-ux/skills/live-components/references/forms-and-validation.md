# Forms and validation

## Rendering a full Symfony form: `ComponentWithFormTrait`

Render an entire Symfony form in a component to get instant validation as the user types. The trait
requires an `instantiateForm()` method that rebuilds the **same** form on each AJAX render — so the
form's underlying data is a `#[LiveProp]`.

```php
use Symfony\UX\LiveComponent\ComponentWithFormTrait;
use Symfony\UX\LiveComponent\DefaultActionTrait;

#[AsLiveComponent]
class PostForm extends AbstractController
{
    use DefaultActionTrait;
    use ComponentWithFormTrait;

    #[LiveProp]
    public ?Post $initialFormData = null;

    protected function instantiateForm(): FormInterface
    {
        return $this->createForm(PostType::class, $this->initialFormData);
    }
}
```

```twig
<div {{ attributes }}>
    {{ form_start(form) }}   {# `form` is provided by the trait #}
        {{ form_row(form.title) }}
        <button>Save</button>
    {{ form_end(form) }}
</div>
```

Render it with `{{ component('PostForm', { initialFormData: post }) }}`, or omit `initialFormData`
(defaults to `null`) for a "new" form.

How it works: the trait holds a writable `$formValues` LiveProp with every field value. When a
field changes, that key updates, an AJAX request re-submits the form, it re-renders with validation
errors for the changed field.

## Submitting via a LiveAction

```php
#[LiveAction]
public function save(EntityManagerInterface $em)
{
    $this->submitForm();          // throws on invalid -> auto re-render with errors
    $post = $this->getForm()->getData();
    $em->persist($post);
    $em->flush();
    return $this->redirectToRoute('app_post_show', ['id' => $post->getId()]);
}
```

Wire the form element to the action (`:prevent` stops the native submit):

```twig
{{ form_start(form, { attr: { 'data-action': 'live#action:prevent', 'data-live-action-param': 'save' } }) }}
```

Key facts about LiveActions + forms:
- When a LiveAction runs, the form is **not yet submitted** and `initialFormData` is **not yet
  updated**. Call `$this->submitForm()` to access the latest data (it's called automatically before
  re-render if you don't).
- To change form data dynamically from an action, mutate `$this->formValues` directly (raw scalar
  POST data) **before** the form submits — e.g. `$this->formValues['title'] = '...'`. For an
  `EntityType` field use the frontend value (the id): `$this->formValues['author'] = $author->getId()`.
  Mutating the entity object after submit has no effect.
- `resetForm()` returns the form to its initial state (instead of redirecting) so it can be reused.

## Submitting via a normal controller

Handle the form in a regular controller; on validation failure pass the `form` into the component so
it renders with errors instead of a fresh form:

```twig
{{ component('PostForm', { initialFormData: post, form: form }) }}
```

## Form rendering gotchas

- **Trailing spaces vanish while typing** on the `input` event because text fields trim. Fix by
  re-rendering `on(change)` or setting `'trim' => false` on the field.
- **`PasswordType` blanks on re-render** because it doesn't refill after submit. Set
  `'always_empty' => false`.

## Collection types

- **`CollectionType`** (`allow_add`/`allow_delete`/`by_reference: false`): add/remove rows from
  LiveActions by mutating `$this->formValues['comments'][] = []` / `unset(...)`. With Doctrine, add
  `orphanRemoval: true` and `cascade={"persist"}` to the relation.
- **`LiveCollectionType`** + `LiveCollectionTrait`: does the same with near-zero code, auto-rendering
  Add/Delete buttons that work out of the box. Customize via `button_add`/`button_delete` options or
  the `live_collection_*` / `live_collection_entry_*` form-theme blocks.

## Validation without the Form component: `ValidatableComponentTrait`

For data not backed by a Symfony form, add constraints to props and use the trait. Add
`#[Assert\Valid]` on a prop whose object should also be validated.

```php
use Symfony\UX\LiveComponent\ValidatableComponentTrait;
use Symfony\Component\Validator\Constraints as Assert;

#[AsLiveComponent]
class EditUser
{
    use ValidatableComponentTrait;

    #[LiveProp(writable: ['email', 'plainPassword'])]
    #[Assert\Valid]
    public User $user;

    #[LiveProp] #[Assert\IsTrue]
    public bool $agreeToTerms = false;
}
```

Validation is **smart**: a field is only validated once its model has been updated on the frontend.
Trigger full validation in an action with `$this->validate()` (throws on failure but still
re-renders). Render errors via the `_errors` variable:

```twig
{% if _errors.has('agreeToTerms') %}<div class="error">{{ _errors.get('agreeToTerms') }}</div>{% endif %}
<input type="checkbox" data-model="agreeToTerms" class="{{ _errors.has('agreeToTerms') ? 'is-invalid' : '' }}">
```

- `resetValidation()` clears errors (e.g. after a successful save when reusing the form);
  `clearValidation()` is used when resetting embedded/new components.
- A validated component "remembers" it was validated, so subsequent edits re-validate.

### Real-time validation timing

Validation runs on the `input` event by default (i.e. as the user types). To validate only after a
field is left, use `on(change)`: `data-model="on(change)|user.email"`.
