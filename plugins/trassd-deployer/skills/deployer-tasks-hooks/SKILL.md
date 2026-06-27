---
name: deployer-tasks-hooks
description: Write custom Deployer tasks and wire them into the deploy flow — task()/desc() definitions, run()/cd() command execution with {{...}} config interpolation, before()/after() hooks, group tasks, and per-task modifiers (once, oncePerNode, limit, select, hidden, disable). Use when adding a custom task to a deploy.php recipe or hooking into the deployment lifecycle (e.g. "add a build step", "run a command after deploy:symlink", "restart php-fpm during deploy").
---

# Deployer: Tasks & Hooks

Deployer recipes are built from **hosts** and **tasks**. Use this skill when defining
custom tasks or attaching them to lifecycle points in a `deploy.php` recipe.

## Defining a task

Use `task()` to define a task and `desc()` to give it a description (shown by `dep list`):

```php
namespace Deployer;

desc('Restart php-fpm');
task('restart-fpm', function () {
    run('sudo systemctl restart php-fpm');
});
```

`desc('…')` set just before `task()` applies to the next task. The equivalent chained
form is `task('restart-fpm', fn() => ...)->desc('Restart php-fpm');`.

Pass only the name to fetch an already-defined task and chain config onto it:

```php
task('deploy:cleanup')->disable();
```

## Running commands

- `run('command')` — runs on the current remote host, returns trimmed stdout.
- `runLocally('command')` — runs on the local machine.
- `cd('{{deploy_path}}')` — changes the working directory for the **rest of the current
  task**. The next task starts fresh.
- `within('{{release_path}}', fn() => ...)` — scoped `cd()` that restores the previous
  directory afterwards. Prefer this over `cd()` when the change should not leak.
- `test('[ -d {{release_path}} ]')` — returns a bool for a shell test.

```php
task('build', function () {
    within('{{release_path}}', function () {
        run('composer install --no-dev');
        run('npm ci && npm run build');
    });
});
```

`run()` named arguments worth knowing: `cwd:` (override working dir for one call only),
`timeout:` (seconds; default `{{default_timeout}}` = 300, `null` disables),
`env:` (per-call environment vars), `nothrow: true` (return output instead of throwing on
non-zero exit), `secrets:` (map `%name%` placeholders to redacted values for safe logs).

## Config interpolation with `{{...}}`

Any `{{name}}` in a command, message, or string is replaced with the config value for the
current host. Set config with `set()`, read it with `get()`, append to array config with
`add()`:

```php
set('keep_releases', 5);
add('shared_files', ['.env']);

task('whoami', function () {
    $user = get('user');           // explicit read
    run('echo {{deploy_path}}');   // inline interpolation
});
```

Dynamic config is a callback, resolved on first access and cached per host:

```php
set('current_branch', fn() => runLocally('git rev-parse --abbrev-ref HEAD'));
```

To read a value that may be overridden on the CLI with `-o name=value`, always use a
callback — a plain `get()` at recipe-load time captures the default and misses `-o`.
Escape a literal brace with a backslash: `run('echo \{{not_replaced}}')`.

## Hooks: before / after / fail

Attach a task (or callback) to run relative to another task:

```php
before('deploy:symlink', 'build');          // run task "build" before symlinking
after('deploy:symlink', 'restart-fpm');     // run a task after
after('deploy:failed', function () {        // inline callback
    run('echo something went wrong');
});
fail('deploy', 'deploy:unlock');            // run on failure of "deploy"
```

`fail()` replaces any previous handler for the same task. The chained equivalents of
`before()`/`after()` are `->addBefore('task')` / `->addAfter('task')`. Inspect attached
hooks with `dep tree <task>`.

## Group tasks

Pass an **array of task names** instead of a callback to run them in order:

```php
task('deploy', [
    'deploy:prepare',
    'deploy:vendors',
    'deploy:symlink',
    'deploy:cleanup',
]);
```

Call another task from inside a callback with `invoke('task-name')`.

## Per-task modifiers

Chain these off `task('name')`:

| Modifier | Effect |
|---|---|
| `once()` | Run on a single host, not all selected hosts. |
| `oncePerNode()` | Run once per node (hosts sharing a hostname count as one node). |
| `limit(int)` | Cap how many hosts run the task concurrently (default: unlimited). |
| `select('selector')` | Restrict the task to hosts matching a selector (replaces any prior selector). |
| `addSelector(...)` | Add another selector clause (OR). |
| `hidden()` | Hide the task from `dep list`. |
| `verbose()` | Always run as if `-v` were passed. |
| `disable()` / `enable()` | Turn a task off / back on. Disabled tasks never run. |

```php
desc('Update OS packages');
task('apt:update', function () {
    run('apt-get update');
})->oncePerNode();

task('migrate', function () {
    run('{{bin/php}} artisan migrate --force');
})->once();                       // run on one host only

task('warm-cache')->select('type=web');   // only on hosts labeled type=web
```

Selectors match host **labels** (set via `->setLabels([...])`). Syntax: `,` is OR, `&` is
AND, `|` is OR-within-a-value, `!=` negates; `all` matches every host and a bare token is
treated as `alias=...`. From PHP, `select('type=web,env=prod')` returns matching hosts and
`on($hosts, fn() => ...)` runs a callback on each.

## Rules of thumb

- One task = one responsibility; compose them with group tasks rather than one giant
  callback.
- Hook into named lifecycle tasks (e.g. `before('deploy:symlink', ...)`) instead of
  redefining `deploy` — your hook survives recipe upgrades.
- Use `once()` for steps that must run a single time (migrations, cache clears) and
  `oncePerNode()` for host-level work (package installs) when aliases share a machine.
- Keep secrets out of logs: pass them through `run(..., secrets: [...])`.
- Use `{{...}}` interpolation and `set()`/`get()` rather than hardcoding paths, so a task
  works across hosts and stages.
