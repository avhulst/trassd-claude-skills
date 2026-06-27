# Quality tooling config samples

Full, copy-ready configs for a single Contao extension (Symfony bundle). They are
distilled from the Contao monorepo configs and adapted to a normal one-bundle
layout (`src/`, `tests/`) with the standard `vendor/bin` runner — not the
monorepo's `vendor-bin/<tool>` bin-plugin layout.

## ecs.php

Uses the `Contao\EasyCodingStandard\Set\SetList::CONTAO` set from
`contao/easy-coding-standard`, the LGPL header comment fixer, parallel runs and a
cache directory. Adapt the header text and paths to your bundle.

```php
<?php

declare(strict_types=1);

use Contao\EasyCodingStandard\Set\SetList;
use PhpCsFixer\Fixer\Comment\HeaderCommentFixer;
use Symplify\EasyCodingStandard\Config\ECSConfig;
use Symplify\EasyCodingStandard\ValueObject\Option;

return ECSConfig::configure()
    ->withSets([SetList::CONTAO])
    ->withPaths([
        __DIR__.'/src',
        __DIR__.'/tests',
    ])
    ->withRootFiles()
    ->withParallel()
    ->withSpacing(Option::INDENTATION_SPACES, "\n")
    ->withConfiguredRule(HeaderCommentFixer::class, [
        'header' => "This file is part of the Acme bundle.\n\n(c) Acme\n\n@license MIT",
    ])
    ->withCache(sys_get_temp_dir().'/ecs/acme')
;
```

If you do not want a license header on every file, drop the
`withConfiguredRule(HeaderCommentFixer::class, ...)` line.

Use `->withSkip([...])` to exclude a fixer/sniff for specific files, e.g.:

```php
->withSkip([
    \PhpCsFixer\Fixer\Whitespace\MethodChainingIndentationFixer::class => [
        '*/DependencyInjection/Configuration.php',
    ],
])
```

## phpstan.neon

Contao core runs PHPStan at `level: 6` with the phpstan-symfony and
phpstan-phpunit extensions plus bleedingEdge. For a single bundle you include the
extension neon files directly from `vendor/`:

```neon
includes:
    - vendor/phpstan/phpstan/conf/bleedingEdge.neon
    - vendor/phpstan/phpstan-symfony/extension.neon
    - vendor/phpstan/phpstan-symfony/rules.neon
    - vendor/phpstan/phpstan-phpunit/extension.neon
    - vendor/phpstan/phpstan-phpunit/rules.neon

parameters:
    level: 6

    paths:
        - src
        - tests

    # Contao magic-property carriers PHPStan should treat as universal object crates
    universalObjectCratesClasses:
        - Contao\BackendUser
        - Contao\Database\Result
        - Contao\Model
        - Contao\Template

    excludePaths:
        - tests/Fixtures/*

    ignoreErrors:
        - identifier: missingType.iterableValue
```

Install the extensions as dev deps:

```bash
composer require --dev phpstan/phpstan phpstan/phpstan-symfony phpstan/phpstan-phpunit
```

## rector.php

Uses the `Contao\Rector\Set\SetList::CONTAO` set from `contao/contao-rector`
plus a PHP-version set. Set the PHP set to match your bundle's minimum PHP.

```php
<?php

declare(strict_types=1);

use Contao\Rector\Set\SetList;
use Rector\Config\RectorConfig;

return RectorConfig::configure()
    ->withPhpSets(php83: true)
    ->withSets([SetList::CONTAO])
    ->withPaths([
        __DIR__.'/src',
        __DIR__.'/tests',
    ])
    ->withRootFiles()
    ->withParallel()
    ->withCache(sys_get_temp_dir().'/rector/acme')
;
```

Install:

```bash
composer require --dev rector/rector contao/contao-rector
```

Use `->withSkip([SomeRector::class, OtherRector::class => ['path/to/File.php']])`
to disable individual rules globally or per file.

## depcheck.php

Uses `shipmonk/composer-dependency-analyser`. It detects unused dependencies,
shadow dependencies (used but not required) and wrong dev/prod placement.

```php
<?php

declare(strict_types=1);

use ShipMonk\ComposerDependencyAnalyser\Config\Configuration;
use ShipMonk\ComposerDependencyAnalyser\Config\ErrorType;

return (new Configuration())
    ->disableReportingUnmatchedIgnores()

    // Classes referenced but provided at runtime by Contao / the app.
    ->ignoreUnknownClasses([
        'Imagick',
    ])

    // A package that is required but only used indirectly (e.g. provides a
    // function or a Twig filter) — silence the "unused dependency" finding.
    ->ignoreErrorsOnPackage('symfony/deprecation-contracts', [ErrorType::UNUSED_DEPENDENCY])
;
```

Install:

```bash
composer require --dev shipmonk/composer-dependency-analyser
```

## composer.json scripts

A combined gate mirroring Contao core's `all` script, adapted to `vendor/bin`:

```json
{
    "scripts": {
        "ecs": "@php vendor/bin/ecs check --fix",
        "phpstan": "@php vendor/bin/phpstan analyze",
        "rector": "@php vendor/bin/rector --dry-run",
        "depcheck": "@php vendor/bin/composer-dependency-analyser --config=depcheck.php",
        "tests": "@php vendor/bin/phpunit",
        "all": [
            "@rector",
            "@ecs",
            "@phpstan",
            "@depcheck",
            "@tests"
        ]
    }
}
```

Run any of them with `composer ecs`, `composer phpstan`, … or the whole gate with
`composer all`. In CI, run `vendor/bin/ecs check` (no `--fix`) and
`vendor/bin/rector --dry-run` so the build fails instead of rewriting files.
