---
name: symfony-service-config
description: >-
  Configure Symfony application services the recommended way — autowiring,
  autoconfiguration, private services, tags, and prefixed parameters, preferring
  attributes over YAML/XML. Use when wiring services, editing
  config/services.yaml, injecting dependencies into controllers/services, or
  deciding between autowired vs. manually-wired arguments and tagged collections.
---

# Symfony Service Config

Configure application services so that Symfony does the wiring for you. Lean on
autowiring + autoconfiguration, keep services private, inject dependencies by
type-hint, and only fall back to explicit config for the arguments the container
genuinely can't figure out.

## The default services.yaml

New projects ship with this config — assume it and build on it. Don't replace
it; add specific service entries only when you need to override something.

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true       # inject dependencies by type-hint
        autoconfigure: true  # auto-tag (commands, subscribers, Twig ext, ...)

    # registers every class in src/ as a service whose id == its FQCN
    App\:
        resource: '../src/'
        exclude:
            - '../src/DependencyInjection/'
            - '../src/Entity/'
            - '../src/Kernel.php'
```

Rules of thumb:

- **One service per class, id = fully-qualified class name.** That FQCN id is
  exactly what autowiring matches against type-hints.
- Order matters: later definitions *replace* earlier ones. Override an imported
  service by re-declaring it by its class-name id below the `App\` import.
- `exclude` paths (entities, DTOs, value objects, the Kernel) are not
  instantiated as services. Excluding also slightly speeds up the `dev`
  container rebuild. You can also use `#[Exclude]` on a single class.
- Unused private classes from `src/` are automatically removed from the compiled
  container, so importing everything is cheap.

## Inject by type-hinting the constructor

This is dependency injection — the preferred way to get a service. Type-hint the
class or interface; autowiring finds the matching service id and passes it.

```php
use Psr\Log\LoggerInterface;

class MessageGenerator
{
    public function __construct(
        private LoggerInterface $logger,
    ) {}
}
```

- Prefer type-hinting **interfaces** over concrete classes. Core aliases make
  this work (e.g. `Psr\Log\LoggerInterface` is aliased to the `logger` service).
- In controllers, type-hint services as **action-method arguments** or
  constructor arguments — do not reach into the container with
  `$container->get()`.
- `php bin/console debug:autowiring [search]` lists every autowireable type.
- Autowiring is compiled, so there's **no runtime overhead** (only a `dev`
  rebuild cost when classes change).

## Scalar / non-service arguments

Autowiring only resolves *objects*. A `string`, `int`, parameter, or env var
must be wired explicitly. Prefer the `#[Autowire]` attribute (config lives next
to the code that uses it):

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;

public function __construct(
    #[Autowire('%kernel.project_dir%/data')] private string $dataDir,
    #[Autowire(param: 'kernel.debug')]       private bool $debugMode,
    #[Autowire(env: 'SOME_ENV_VAR')]         private string $senderName,
    #[Autowire(service: 'monolog.logger.request')] private LoggerInterface $logger,
) {}
```

Alternatives when you'd rather keep it out of the class:

- **Per-service argument** in YAML: `arguments: { $adminEmail: 'manager@example.com' }`.
  Renaming the argument gives a clear compile-time error, so it isn't fragile.
- **`bind`** under `_defaults` to set an argument for *every* service in the
  file, matched by name (`$adminEmail`), type, or both
  (`string $adminEmail`). See [references/binding-and-arguments.md](references/binding-and-arguments.md).

For choosing among multiple implementations of one interface, use a named
autowiring alias or `#[Target]` rather than wiring each injection point —
see [references/binding-and-arguments.md](references/binding-and-arguments.md).

## Tags: prefer autoconfiguration

Tags tell Symfony (or a bundle) to treat a service specially (e.g.
`twig.extension`). With `autoconfigure: true` you almost never write tags by
hand:

- Implementing a known interface (e.g. `Twig\Extension\ExtensionInterface`)
  auto-applies its tag.
- Attributes like `#[AsMessageHandler]`, `#[AsEventListener]`, `#[AsCommand]`
  register the right tag automatically.

Apply this to your *own* base types with `#[AutoconfigureTag]` on the interface,
or `_instanceof` in YAML — both beat repeating a tag on every implementation:

```php
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;

#[AutoconfigureTag('app.custom_tag')]
interface CustomInterface {}
```

Reach for a **manual tag** only when no autoconfiguration rule fits, or when the
tag needs per-service attributes (priority, index, alias). To consume a tag,
inject the collection with `#[AutowireIterator('app.handler')]` /
`!tagged_iterator` instead of writing a compiler pass. Details, priority/index
options, and the compiler-pass fallback are in
[references/tags.md](references/tags.md).

## Parameters: short and prefixed

- Put **application config** (values that don't change per machine, e.g. a
  feature toggle or notification sender) in `parameters:` in
  `config/services.yaml`; override per environment in `services_dev.yaml` /
  `services_prod.yaml`.
- Put **infrastructure config** (differs per machine: DB URL, API host) in
  **env vars**, and **secrets** for sensitive values — not in parameters.
- Prefix your parameter names with `app.` to avoid collisions; use one or two
  meaningful words. Be consistent with the separator.

```yaml
parameters:
    app.contents_dir: '%kernel.project_dir%/var/contents'
    app.admin_email: 'admin@example.com'
```

- Reference a service (not a string) with a leading `@`: `'@logger'` (escape a
  literal `@` as `@@`). Reference a parameter with `%name%`.
- Options that **rarely change** (e.g. items-per-page) are better as PHP
  **constants** on the relevant class — usable in Twig and entities too, not
  just where the container is available.

## Public vs. private services

- Services are **private by default** — keep them that way. A private service
  can't be fetched via `$container->get()`, which forces proper dependency
  injection.
- Only make a service `public: true` (or `#[Autoconfigure(public: true)]`) when
  something genuinely needs to pull it from the container. For lazy fetching,
  prefer a **service locator** over a public service.

## Format & altitude

- **Prefer attributes** (`#[Autowire]`, `#[AsEventListener]`,
  `#[AutoconfigureTag]`, routing/security attributes) — config lives with the
  code and uses one format.
- When you *must* use a config file, **YAML** is the recommended format for your
  own services (concise, newcomer-friendly); XML and PHP are also supported.
- Use **autowiring + autoconfiguration** for your application. **Public/reusable
  bundles** must wire services explicitly instead — they don't control the host
  application's container.
- Run `php bin/console lint:container` (e.g. in CI) to catch misconfiguration
  before deploy.

## References

- [references/binding-and-arguments.md](references/binding-and-arguments.md) —
  explicit arguments, `bind`, choosing a specific service, multiple
  implementations (aliases, named aliases, `#[Target]`), env-limited services.
- [references/tags.md](references/tags.md) — autoconfiguring tags, manual tags
  with attributes, injecting tagged collections (priority/index), compiler-pass
  fallback.
