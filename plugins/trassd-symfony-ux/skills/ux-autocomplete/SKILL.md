---
name: ux-autocomplete
description: Add autocompleting, Ajax-backed form fields with Symfony UX Autocomplete (Tom Select). Use when an EntityType/ChoiceType/EnumType or any <select> should load and filter choices on the fly instead of rendering all options up front, when building a reusable Ajax entity field, or when securing/customizing an autocomplete endpoint.
---

# Symfony UX Autocomplete

Turns an `EntityType`, `ChoiceType`, `EnumType`, `TextType`, or any plain
`<select>` into a [Tom Select](https://tom-select.js.org/)-powered smart UI
control via a Stimulus controller. Install with
`composer require symfony/ux-autocomplete`. Requires StimulusBundle to be
configured; with WebpackEncore also rebuild assets (not needed with
AssetMapper).

## Choosing local vs. Ajax

- **Local (no Ajax):** all options render on the page and are searched in the
  browser. Just add `'autocomplete' => true` to the field. Good for small,
  bounded choice lists.
- **Ajax-powered:** choices are fetched from a server endpoint as the user
  types. Use this for `EntityType` with **many** rows, or when you need to
  search on fields beyond the displayed label. Build a dedicated form-type
  class (below) — do **not** just flip a flag.

Rule of thumb: if the option count is unbounded or large, go Ajax.

## Local autocomplete (no Ajax)

Add one option to an existing `ChoiceType`/`EntityType`/`TextType`:

```php
$builder->add('food', EntityType::class, [
    'class' => Food::class,
    'placeholder' => 'What should we eat?',
    'autocomplete' => true,
]);
```

The Stimulus controller transforms the rendered `<select>` automatically.

## Ajax-powered entity field (preferred for large entities)

Create a reusable form type. Three requirements:

1. The class carries `#[AsEntityAutocompleteField]`.
2. `getParent()` returns `BaseEntityAutocompleteType`.
3. Configure normal `EntityType` options plus the Ajax-specific options inside
   `configureOptions()`.

```php
use Symfony\UX\Autocomplete\Form\AsEntityAutocompleteField;
use Symfony\UX\Autocomplete\Form\BaseEntityAutocompleteType;

#[AsEntityAutocompleteField]
class FoodAutocompleteField extends AbstractType
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'class' => Food::class,
            'placeholder' => 'What should we eat?',
            'searchable_fields' => ['name'],   // omit/null = search ALL fields
            'security' => 'ROLE_FOOD_ADMIN',   // secure the endpoint
        ]);
    }

    public function getParent(): string
    {
        return BaseEntityAutocompleteType::class;
    }
}
```

Use it in a form with **no third argument** to `add()`:

```php
$builder->add('food', FoodAutocompleteField::class);
```

**Critical:** options passed as the 3rd argument of `->add()` are *not*
preserved during the Ajax fetch. Put all options inside the field class, or use
`extra_options` (see below). `make:autocomplete-field` (MakerBundle) scaffolds
this class.

## The `#[AsEntityAutocompleteField]` attribute

This attribute only accepts two constructor arguments — it does **not** carry
the form options (those live in `configureOptions()`):

- `alias` (default: derived from the class short name) — the registry key /
  route wildcard for this autocompleter.
- `route` (default: `ux_entity_autocomplete`) — the route used for the Ajax
  call. Override it to point at a custom-firewalled route, e.g.
  `#[AsEntityAutocompleteField(route: 'ux_entity_autocomplete_admin')]`.

## Option reference

These apply to `ChoiceType`/`EntityType`/`TextType` (local **or** Ajax):

- `autocomplete` (default `false`) — activate the Stimulus controller.
- `tom_select_options` (default `[]`) — pass raw Tom Select options, including
  `plugins`. See [references/tom-select.md](references/tom-select.md).
- `options_as_html` (default `false`) — set `true` if option/`text` values
  contain HTML; otherwise values are escaped to prevent XSS.
- `autocomplete_url` (default `null`) — point a field at a custom Ajax endpoint
  (e.g. a custom `ChoiceType`).
- `loading_more_text`, `no_results_found_text`, `no_more_results_text` —
  translated UI strings (domain `AutocompleteBundle`).

These apply **only** to Ajax field classes (`getParent()` →
`BaseEntityAutocompleteType`), verified against the docs:

- `searchable_fields` (default `null`) — array of entity fields to search;
  `null` searches all. Relation fields like `category.name` are allowed.
- `security` (default `false`) — `false` = open endpoint; a role string (e.g.
  `'ROLE_FOO'`); or a `callable(Security): bool` returning grant/deny.
- `filter_query` (default `null`) — `callable(QueryBuilder, string $query,
  EntityRepository)` for full control of the search query. **Incompatible with
  `searchable_fields` and `max_results`.**
- `max_results` (default `10`), `min_characters` (default `3`).
- `preload` (default `'focus'`) — `'focus'`, `true` (load on init), or `false`.
- `extra_options` (default `[]`) — see below.

## Securing the results endpoint

The Ajax endpoint is open by default. Always set `security` when results expose
non-public data:

```php
'security' => 'ROLE_FOO',
// or a callback:
'security' => fn (Security $security): bool => $security->isGranted('ROLE_FOO'),
```

For a custom autocompleter (no form), enforce access in `isGranted(Security)`.

## Extra options (state across the Ajax round-trip)

Field options are lost on the Ajax call, so things like "exclude the current
record" can't rely on closures capturing request state. Pass scalars/arrays via
`extra_options` instead; they are re-sent on every Ajax call (and checksum-signed
server-side, so they cannot be tampered with). Read them back inside a default
`query_builder`. Full example:
[references/extra-options.md](references/extra-options.md).

## Customizing rendering & extending Tom Select

- Simple tweaks: pass `tom_select_options`. Bootstrap styling: toggle the
  Tom Select CSS in `assets/controllers.json`.
- JS-level options (e.g. `onInitialize`, `onChange`): write a custom Stimulus
  controller that listens for `autocomplete:pre-connect` (mutate
  `event.detail.options` before init) and `autocomplete:connect` (access
  `event.detail.tomSelect`). Load it **eagerly** and attach it via a
  `data-controller` attr in the field's `attr` option. The core controller is
  `symfony/ux-autocomplete/autocomplete`. See
  [references/tom-select.md](references/tom-select.md).

## Custom / non-entity autocompleters

- **Doctrine entity, no form:** implement `EntityAutocompleterInterface`
  (`getEntityClass`, `createFilteredQueryBuilder`, `getLabel`, `getValue`,
  `isGranted`) and tag the service `ux.entity_autocompleter` with an `alias`
  (e.g. via `#[AutoconfigureTag('ux.entity_autocompleter', ['alias' => 'food'])]`).
  Reach it at the `ux_entity_autocomplete` route with that alias. Implement
  `OptionsAwareEntityAutocompleterInterface` to receive `extra_options`.
- **Non-entity:** build your own endpoint returning JSON
  `{"results": [{"value": "...", "text": "..."}]}` (the search term arrives as
  the `query` parameter), then pass its URL via `autocomplete_url` or the
  controller's `url` value.
- **Manual Stimulus use:** attach
  `{{ stimulus_controller('symfony/ux-autocomplete/autocomplete', { url: path(...) }) }}`
  to a `<select>`/`<input>` directly. See
  [references/custom-autocompleter.md](references/custom-autocompleter.md).

## Testing

In a `TypeTestCase`, register `AutocompleteChoiceTypeExtension` via
`getTypeExtensions()` so the `autocomplete` options resolve.

## Checklist

- Large/unbounded entity → Ajax field class, not just `'autocomplete' => true'`.
- All field options live in the field class (or `extra_options`), never the
  `->add()` 3rd argument.
- `security` is set whenever results aren't public.
- `options_as_html` only when output is trusted/escaped intentionally.
- `filter_query` not combined with `searchable_fields`/`max_results`.
