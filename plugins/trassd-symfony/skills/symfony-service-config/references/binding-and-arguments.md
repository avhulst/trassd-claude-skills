# Wiring arguments explicitly

Use these only for the arguments autowiring can't resolve on its own. Everything
else stays autowired.

## Per-service scalar / service arguments

Set just the problematic argument; the rest remain autowired.

```yaml
# config/services.yaml
services:
    App\Service\SiteUpdateManager:
        arguments:
            $adminEmail: 'manager@example.com'
```

Pass a **specific service** (not a string) with a leading `@`. This overrides
autowiring for that one argument:

```yaml
services:
    App\Service\MessageGenerator:
        arguments:
            # pass the service whose id is 'monolog.logger.request'
            $logger: '@monolog.logger.request'
```

Scalars, constants and collections can be passed directly:

```yaml
services:
    App\Service\SomeService:
        arguments:
            - 'Foo'                 # string
            - true                  # bool
            - 7                     # int
            - !php/const PDO::FETCH_NUM   # constant / enum case
            - '@logger'             # service reference
            - { first: true, second: 'Foo' }  # collection
```

Renaming a wired argument (e.g. `$adminEmail` -> `$mainEmail`) produces a clear
exception on the next request, so explicit wiring by name is not fragile.

## bind: one value for many services

Put `bind` under `_defaults` to apply an argument to *every* service defined in
that file (including controller arguments). Match by name, type, or both.

```yaml
# config/services.yaml
services:
    _defaults:
        bind:
            $adminEmail: 'manager@example.com'         # by name
            Psr\Log\LoggerInterface: '@monolog.logger.request'  # by type
            string $adminEmail: 'manager@example.com'  # name + type
            Psr\Log\LoggerInterface $requestLogger: '@monolog.logger.request'
            iterable $rules: !tagged_iterator app.foo.rule
```

`bind` can also be applied to a specific service or when loading many services
at once.

## Choosing among multiple implementations of one interface

If two services implement the same interface, autowiring can't pick one — it
matches a service id to the type-hint, and now there are two candidates. Resolve
it with aliases.

```yaml
# config/services.yaml
services:
    App\Util\Rot13Transformer: ~
    App\Util\UppercaseTransformer: ~

    # default: any TransformerInterface type-hint gets Rot13Transformer
    App\Util\TransformerInterface: '@App\Util\Rot13Transformer'

    # named autowiring alias: a $shoutyTransformer argument gets UppercaseTransformer
    App\Util\TransformerInterface $shoutyTransformer: '@App\Util\UppercaseTransformer'
```

So `private TransformerInterface $transformer` receives `Rot13Transformer`,
while `private TransformerInterface $shoutyTransformer` receives
`UppercaseTransformer`.

When only one service implements an interface (under resource/prototype
loading), Symfony creates the alias automatically — you don't need to declare
it.

### `#[Target]` — pick the implementation from code

`#[Target]` references the **argument name used in the named alias**, so the
local property name can differ from any implementation name (and a typo throws):

```php
use App\Util\TransformerInterface;
use Symfony\Component\DependencyInjection\Attribute\Target;

class MastodonClient
{
    public function __construct(
        #[Target('shoutyTransformer')]
        private TransformerInterface $transformer,
    ) {}
}
```

`#[Target]` accepts only the named-alias argument name — not a service id or
alias. The string is normalized to camelCase, so variants like
`shouty.transformer` also resolve. List named aliases with
`php bin/console debug:autowiring TransformerInterface`.

## Aliases to enable autowiring

Autowiring needs a service whose id equals the type-hint. If a service's id is
*not* its class name (e.g. `app.rot13.transformer`), add an alias so the class
(or interface) can still be autowired:

```yaml
services:
    app.rot13.transformer:
        class: App\Util\Rot13Transformer

    # makes App\Util\Rot13Transformer type-hints resolve to that service
    App\Util\Rot13Transformer: '@app.rot13.transformer'
```

## Limiting a service to an environment

```php
use Symfony\Component\DependencyInjection\Attribute\When;
use Symfony\Component\DependencyInjection\Attribute\WhenNot;

#[When(env: 'dev')]
class SomeClass {}

#[WhenNot(env: 'dev')]
class AnotherClass {}  // registered in every env except dev
```

Note: a `when@<env>` YAML block has its own scope and does **not** inherit
`_defaults` from the main `services` section — redefine `_defaults` (autowire,
autoconfigure) inside each `when@<env>` block that needs it.
