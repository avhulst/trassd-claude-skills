# Custom autocompleters (no form) and manual Stimulus use

## Doctrine entity autocompleter without the form system

Create a service implementing `EntityAutocompleterInterface` and tag it
`ux.entity_autocompleter` with an `alias`:

```php
use Symfony\Bundle\SecurityBundle\Security;
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;
use Symfony\UX\Autocomplete\EntityAutocompleterInterface;

#[AutoconfigureTag('ux.entity_autocompleter', ['alias' => 'food'])]
class FoodAutocompleter implements EntityAutocompleterInterface
{
    public function getEntityClass(): string { return Food::class; }

    public function createFilteredQueryBuilder(EntityRepository $repository, string $query): QueryBuilder
    {
        return $repository->createQueryBuilder('food')
            ->andWhere('food.name LIKE :search OR food.description LIKE :search')
            ->setParameter('search', '%'.$query.'%');
    }

    public function getLabel(object $entity): string { return $entity->getName(); }
    public function getValue(object $entity): string { return $entity->getId(); }
    public function isGranted(Security $security): bool { return true; } // enforce access here
}
```

The endpoint is reachable at the `ux_entity_autocomplete` route with the alias:

```twig
{{ path('ux_entity_autocomplete', { alias: 'food' }) }}
```

To receive `extra_options`, implement
`OptionsAwareEntityAutocompleterInterface` and store the value passed to
`setOptions()` (see ../references/extra-options.md).

## Customizing the Ajax route

The default route is `/autocomplete/{alias}/`. To put it behind a specific
firewall, declare a route pointing at the controller service and reference it
from the attribute:

```yaml
# config/routes/attributes.yaml
ux_entity_autocomplete_admin:
    controller: ux.autocomplete.entity_autocomplete_controller
    path: '/admin/autocomplete/{alias}'
```

```php
#[AsEntityAutocompleteField(route: 'ux_entity_autocomplete_admin')]
class FoodAutocompleteField { /* ... */ }
```

## Non-entity endpoint (JSON contract)

For anything that is not a Doctrine entity, build your own endpoint. The search
term arrives as the `query` query parameter. The response must be:

```json
{
  "results": [
    { "value": "1", "text": "Pizza" },
    { "value": "2", "text": "Banana" }
  ]
}
```

For Tom Select option groups:

```json
{
  "results": {
    "options": [
      { "value": "1", "text": "Pizza", "group_by": ["food"] }
    ],
    "optgroups": [{ "value": "food", "label": "food" }]
  }
}
```

Pass the URL via the form field's `autocomplete_url` option, or to the
controller's `url` value.

## Manual Stimulus controller (outside Forms)

Attach the core controller to a `<select>`/`<input>` directly:

```twig
<select name="food"
    {{ stimulus_controller('symfony/ux-autocomplete/autocomplete') }}>
</select>
```

For Ajax-backed options, pass a `url` value:

```twig
<select name="food"
    {{ stimulus_controller('symfony/ux-autocomplete/autocomplete', {
        url: path('ux_entity_autocomplete', { alias: 'food' })
    }) }}>
</select>
```
