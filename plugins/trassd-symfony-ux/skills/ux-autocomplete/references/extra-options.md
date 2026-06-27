# Passing extra options across the Ajax round-trip

Field options are **not** preserved when the field is re-rendered during an Ajax
call. So a `query_builder` closure that captures request state (e.g. the current
record's id) works on the first render but is dropped on subsequent Ajax fetches.

`extra_options` fixes this: its values are re-sent on every Ajax call. Only
scalars (`string`, `int`, `float`, `bool`), `null`, and arrays of those are
allowed. The values are checksum-signed server-side, so a tampered request is
rejected.

## In the form: pass the extra option

```php
class FoodForm extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $currentFoodId = $builder->getData()->getId();

        $builder->add('food', FoodAutocompleteField::class, [
            'extra_options' => [
                'excluded_foods' => [$currentFoodId],
            ],
        ]);
    }
}
```

## In the field class: read it back in `query_builder`

```php
#[AsEntityAutocompleteField]
class FoodAutocompleteField extends AbstractType
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'class' => Food::class,
            'query_builder' => function (Options $options) {
                return function (EntityRepository $er) use ($options) {
                    $qb = $er->createQueryBuilder('o');
                    $excluded = $options['extra_options']['excluded_foods'] ?? [];
                    if ([] !== $excluded) {
                        $qb->andWhere($qb->expr()->notIn('o.id', $excluded));
                    }
                    return $qb;
                };
            },
        ]);
    }

    public function getParent(): string
    {
        return BaseEntityAutocompleteType::class;
    }
}
```

## For a custom autocompleter (no form)

Implement `OptionsAwareEntityAutocompleterInterface`, store the array given to
`setOptions(array $options)`, and read `$this->options['extra_options']` inside
`createFilteredQueryBuilder()`.
