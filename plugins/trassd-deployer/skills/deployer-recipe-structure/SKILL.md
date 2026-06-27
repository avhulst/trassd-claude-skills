---
name: deployer-recipe-structure
description: >-
  Structure a Deployer recipe the recommended way — the `namespace Deployer`
  header, importing framework recipes, declaring config with `set()`, and
  choosing between PHP, YAML, and MAML formats. Triggers when creating or
  editing deploy.php, deploy.yaml, or deploy.maml, or running `dep init`.
---

# Deployer Recipe Structure

A **recipe** is the file Deployer loads to define your [hosts](../deployer-hosts/SKILL.md)
and tasks. By default `dep` loads `deploy.php` or `deploy.maml` from the current
directory; point it elsewhere with `-f` / `--file`. Generate a starter recipe
with `dep init` and answer the prompts — Deployer scaffolds a `deploy.php` or
`deploy.yaml` for you.

## The PHP recipe header

Every PHP recipe opens with the `Deployer` namespace. This is what makes the
global functions (`host()`, `task()`, `set()`, `run()`, `import()`, …)
available unqualified.

```php
namespace Deployer;

host('deployer.org');

task('my_task', function () {
    run('whoami');
});
```

Omitting the namespace is the most common recipe mistake — the helper functions
will not resolve.

## Importing framework recipes

Framework recipes (Laravel, Symfony, …) extend the built-in `common` recipe,
which provides the release/symlink deployment flow. Pull one in with
`require`:

```php
namespace Deployer;

require 'recipe/laravel.php';

set('repository', 'git@github.com:example/example.com.git');

host('example.com')
    ->set('remote_user', 'deployer')
    ->set('deploy_path', '~/example');
```

Built-in `recipe/*` and `contrib/*` paths resolve via PHP's include path, so you
do not need `__DIR__` or absolute paths.

The `import()` function is the cross-format equivalent: it `require`s `.php`
files and parses/applies `.maml` and `.yaml` files. Use it to mix formats —
e.g. keep closures in PHP and import them from a YAML or MAML recipe, or pull a
declarative recipe into a PHP one:

```php
import('deploy.maml');     // load a MAML recipe from PHP
import('inventory.yaml');  // load host definitions from YAML
```

## Declaring config with `set()`

`set($name, $value)` declares global config that every host inherits. A host can
override any key with its own `->set()`.

```php
set('keep_releases', 5);
set('repository', 'git@github.com:example/example.com.git');

host('deployer.org');                       // inherits keep_releases = 5
host('medv.io')->set('keep_releases', 10);  // overrides for this host
```

A **callback value** is evaluated lazily on first access and cached per host —
use it for anything computed at runtime, and for any value you want to be
overridable from the CLI with `-o`:

```php
set('whoami', function () {
    return run('whoami');
});
```

Rule of thumb: if a config derives from another config that a user might
override with `-o`, wrap it in a callback. A plain `get()` at recipe load time
captures the default and never sees `-o` overrides.

Reference config inside commands and strings with `{{...}}` interpolation:

```php
task('build', function () {
    cd('{{release_path}}');
    run('npm ci');
    run('npm run build');
});
```

## Choosing PHP, YAML, or MAML

All three formats describe the same recipe; pick by what the recipe needs.

- **PHP (`deploy.php`)** — the full API. Required for closures, custom task
  logic, dynamic `set()` callbacks, and conditional logic. This is what
  `dep init` produces by default and the right default choice.
- **YAML (`deploy.yaml`)** — declarative `import` / `config` / `hosts` /
  `tasks` / `before` / `after`. Validated against Deployer's `schema.json`.
  Good when the recipe is pure configuration with no PHP.
- **MAML (`deploy.maml`)** — a JSON superset with comments, raw multiline
  strings, optional commas, and unquoted keys. Same top-level sections as YAML
  plus `fail`, with friendlier syntax for embedded scripts. Pick `maml` when
  prompted by `dep init`.

A YAML recipe:

```yaml
import:
  - recipe/laravel.php

config:
  repository: "git@github.com:example/example.com.git"
  remote_user: deployer

hosts:
  example.com:
    deploy_path: "~/example"

tasks:
  build:
    - cd: "{{release_path}}"
    - run: "npm run build"

after:
  deploy:failed: deploy:unlock
```

The same recipe in MAML:

```maml
{
  import: ["recipe/laravel.php"]

  config: {
    repository: "git@github.com:example/example.com.git"
  }

  hosts: {
    "example.com": {
      remote_user: "deployer"
      deploy_path: "~/example"
    }
  }

  tasks: {
    build: [
      { cd: "{{release_path}}" }
      { run: "npm run build" }
    ]
  }

  after: { "deploy:failed": "deploy:unlock" }
}
```

Neither YAML nor MAML `config` accepts PHP closures — for runtime-evaluated
values, import a `.php` recipe and `set()` from there. Because all formats can
import each other, the practical pattern is a declarative YAML/MAML recipe that
imports a small `.php` file for the closures and custom tasks it needs.

## Installation modes

How you install Deployer affects how the recipe is invoked, not its structure.

- **Global** (`composer global require deployer/deployer`, or `phive install
  deployer`) — recommended for everyday use; run `dep` from anywhere.
- **Project** (`composer require --dev deployer/deployer`) — pins the version
  per project; preferred for CI/CD. Run `vendor/bin/dep` (alias it to `dep`).
- **Phar** — commit `deployer.phar` to lock the version across local and CI;
  run `php deployer.phar init`. No Composer required.
