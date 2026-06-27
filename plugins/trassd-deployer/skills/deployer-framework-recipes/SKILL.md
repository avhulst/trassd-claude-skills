---
name: deployer-framework-recipes
description: Deploy a PHP framework with Deployer's bundled recipe (Symfony, Laravel, Shopware, etc.) — requiring the recipe, overriding its shared_dirs/shared_files/writable_dirs defaults, and using its framework deploy tasks (deploy:cache:clear, database:migrate, artisan:*) plus bin/console / bin/artisan / bin/php config. Use when deploying a framework app with Deployer or customizing a framework recipe in deploy.php.
---

# Deploying a Framework with a Bundled Recipe

Deployer ships a recipe per framework. Require it at the top of `deploy.php` and
you get a complete `deploy` task tailored to that framework, built on top of the
[common](references/common-recipe.md) recipe.

```php
require 'recipe/symfony.php';   // or recipe/laravel.php, recipe/shopware.php, …
```

Every framework recipe `require`s `recipe/common.php` internally, so the
`common` configuration (`repository`, `keep_releases`, `deploy_path`,
`bin/php`, `bin/git`, …) and the shared deploy stages are always available.

## Rule of thumb: don't reinvent the deploy task

Use the framework recipe's `deploy` task as-is. It already wires the right order
of stages. Customize through configuration overrides and hooks, not by
redefining `deploy`.

The Symfony `deploy` task runs:

```
deploy:prepare → deploy:vendors → deploy:cache:clear → deploy:publish
```

The Laravel `deploy` task runs:

```
deploy:prepare → deploy:vendors → artisan:storage:link → artisan:optimize
→ artisan:migrate → deploy:publish → artisan:reload
```

`deploy:prepare` and `deploy:publish` are group tasks from `common`:
`deploy:prepare` runs info → setup → lock → release → update_code → env →
shared → writable; `deploy:publish` runs symlink → unlock → cleanup → success.

## Overriding shared / writable defaults

Each recipe sets sensible defaults for `shared_dirs`, `shared_files`, and
`writable_dirs`. These override the empty defaults from `recipe/deploy/shared.php`
and `recipe/deploy/writable.php`. Override them again **after** the `require` to
add your own paths.

Recipe defaults:

| Config | Symfony | Laravel |
| --- | --- | --- |
| `shared_dirs` | `['var/log']` | `['storage']` |
| `shared_files` | `['.env.local']` | `['.env']` |
| `writable_dirs` | `['var', 'var/cache', 'var/log', 'var/sessions']` | `['bootstrap/cache', 'storage']` |

Laravel additionally sets `writable_recursive` to `true`.

To extend rather than replace, use `add()`:

```php
require 'recipe/symfony.php';

add('shared_files', ['config/secrets/prod/prod.decrypt.private.php']);
add('shared_dirs', ['public/uploads']);
```

Use `set()` when you want to fully replace the recipe default:

```php
set('writable_dirs', ['var', 'var/cache', 'var/log']);
```

`shared_dirs` / `shared_files` are symlinked from each release into the
`shared/` directory so their contents survive across deploys; `writable_dirs`
are made writable by the web server.

## Framework deploy tasks and where to hook them

The recipes define framework-specific tasks you can add to the pipeline with
`before()` / `after()`.

**Symfony** tasks (in `recipe/symfony.php`):

- `deploy:cache:clear` — clears cache. It only runs `cache:clear` if
  `composer_options` contains `--no-scripts` (otherwise Composer scripts already
  warm the cache).
- `database:migrate` — runs `doctrine:migrations:migrate --allow-no-migration`.
  Set `migrations_config` to point at a non-default migrations configuration.
- `doctrine:schema:validate` — validates the Doctrine mapping. Tune with
  `doctrine_schema_validate_config`.
- `deploy:dump-env` — runs `composer dump-env` to optimize env variables.

```php
require 'recipe/symfony.php';

after('deploy:vendors', 'database:migrate');
```

**Laravel** tasks (in `recipe/laravel.php`) are all `artisan:*` wrappers, e.g.
`artisan:migrate` (`migrate --force`), `artisan:optimize`, `artisan:storage:link`,
`artisan:config:cache`, `artisan:route:cache`, `artisan:queue:restart`,
`artisan:horizon:*`, `artisan:octane:*`. Many take a min/max Laravel version and
skip automatically if `.env` is missing.

```php
require 'recipe/laravel.php';

after('artisan:migrate', 'artisan:db:seed');
```

## bin/console, bin/artisan, bin/php

Framework tasks invoke the framework CLI through configurable bin paths:

- `bin/php` (from `common`) resolves to `/usr/bin/php{{php_version}}` when the
  host has `php_version`, otherwise `which('php')`.
- Symfony: `bin/console` defaults to
  `{{bin/php}} {{release_or_current_path}}/bin/console`; `console_options`
  defaults to `--no-interaction`.
- Laravel: `bin/artisan` defaults to `{{release_or_current_path}}/artisan`.

Override these when your binary lives elsewhere or you pin a PHP version:

```php
host('example.org')
    ->set('php_version', '8.3');           // bin/php → /usr/bin/php8.3

set('bin/console', '{{bin/php}} {{release_or_current_path}}/app/console');
```

Recipes also detect framework versions on first access (`symfony_version`,
`laravel_version`) by parsing `--version` output, and set `log_files`
(`var/log/*.log` for Symfony, `storage/logs/*.log` for Laravel) used by the
`logs:app` task.

## Other framework recipes

The same pattern applies to every bundled recipe — `require 'recipe/<name>.php'`,
then override config and hook tasks. See [All Recipes](references/all-recipes.md)
for the full list (Shopware, Contao, Magento 2, CakePHP, Drupal, TYPO3,
WordPress, Yii2, and more). Each is based on `common`.
