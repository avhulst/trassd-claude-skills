---
name: contao-quality-tooling
description: >-
  Set up and run the Contao quality gate for an extension — Easy Coding Standard
  via contao/easy-coding-standard, PHPStan, Rector, composer dependency analysis,
  and the composer scripts that drive them. Triggers when adding ecs.php /
  phpstan.neon / rector.php to a bundle or running the QA tooling.
---

# Contao quality tooling

Contao is built on Symfony and follows the Symfony Coding Standards very closely.
For reusable bundles, one exception applies: services are prefixed with the
bundle alias rather than named by FQCN (Symfony reusable-bundle best practice),
though controllers may still use their FQCN.

Wire up these tools per extension and chain them in a composer `all` script so
the gate runs with one command. Full copy-ready configs are in
[references/config-samples.md](references/config-samples.md).

## Runner layout: use `vendor/bin`, not the monorepo layout

For a normal one-bundle extension, install each tool as a dev dependency and run
it via `vendor/bin/<tool>`. The Contao **core monorepo** isolates tools with the
`bamarni/composer-bin-plugin` and runs them via `vendor-bin/<tool>/vendor/bin/<tool>`
(e.g. `vendor-bin/ecs/vendor/bin/ecs`). Do **not** copy that layout into a single
bundle — it is core-specific. Use `vendor/bin` throughout.

## ECS — contao/easy-coding-standard

Combines sniffs and fixers that auto-apply the expected Contao/Symfony syntax.

```bash
composer require --dev contao/easy-coding-standard
vendor/bin/ecs check          # report violations
vendor/bin/ecs check --fix    # apply fixes
```

Create `ecs.php` with `ECSConfig::configure()` and the
`Contao\EasyCodingStandard\Set\SetList::CONTAO` set:

```php
use Contao\EasyCodingStandard\Set\SetList;
use Symplify\EasyCodingStandard\Config\ECSConfig;

return ECSConfig::configure()
    ->withSets([SetList::CONTAO])
    ->withPaths([__DIR__.'/src', __DIR__.'/tests'])
    ->withParallel()
    ->withCache(sys_get_temp_dir().'/ecs/acme')
;
```

Add the LGPL/license header via
`->withConfiguredRule(HeaderCommentFixer::class, ['header' => "..."])` and use
`->withRootFiles()` to also lint root-level PHP (like `ecs.php` itself), as Contao
core does. Full sample with `withSkip`/`withSpacing` is in the reference.

## PHPStan

Contao core analyzes at **level 6** with the `phpstan-symfony` and
`phpstan-phpunit` extensions (plus `bleedingEdge`).

```bash
composer require --dev phpstan/phpstan phpstan/phpstan-symfony phpstan/phpstan-phpunit
vendor/bin/phpstan analyze
```

`phpstan.neon` essentials: `includes` the extension `.neon` files from `vendor/`,
`level: 6`, `paths` of `src`/`tests`, and `universalObjectCratesClasses` for
Contao's magic-property carriers (`Contao\Model`, `Contao\Template`,
`Contao\Database\Result`, `Contao\BackendUser`). See the reference for the full file.

## Rector

Automated refactoring + Contao-specific upgrade rules.

```bash
composer require --dev rector/rector contao/contao-rector
vendor/bin/rector              # apply changes
vendor/bin/rector --dry-run    # preview only (use in CI)
```

`rector.php` uses `Contao\Rector\Set\SetList::CONTAO` plus a PHP-version set via
`->withPhpSets(php83: true)` (match your bundle's minimum PHP). Contao core pairs
`->withSets([SetList::CONTAO])` with `->withPhpSets(...)` and skips individual
rules through `->withSkip([...])`.

## Dependency analysis

`shipmonk/composer-dependency-analyser` finds unused deps, shadow deps (used but
not required) and wrong dev/prod placement.

```bash
composer require --dev shipmonk/composer-dependency-analyser
vendor/bin/composer-dependency-analyser --config=depcheck.php
```

`depcheck.php` returns a `Configuration` object; silence unavoidable findings with
`->ignoreUnknownClasses([...])` and `->ignoreErrorsOnPackage('pkg', [ErrorType::UNUSED_DEPENDENCY])`.

## Combined composer gate

Mirror Contao core's `all` script so the whole gate is one command. Core's `all`
chains rector, cs-fixer (ecs), phpstan, depcheck and the test suites. A bundle
equivalent:

```json
"scripts": {
    "ecs": "@php vendor/bin/ecs check --fix",
    "phpstan": "@php vendor/bin/phpstan analyze",
    "rector": "@php vendor/bin/rector --dry-run",
    "depcheck": "@php vendor/bin/composer-dependency-analyser --config=depcheck.php",
    "tests": "@php vendor/bin/phpunit",
    "all": ["@rector", "@ecs", "@phpstan", "@depcheck", "@tests"]
}
```

Run `composer all` locally; in CI use the non-mutating forms
(`vendor/bin/ecs check`, `vendor/bin/rector --dry-run`) so the build fails rather
than silently rewriting files.
