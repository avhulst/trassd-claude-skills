# The `common` Recipe

Every framework recipe `require`s `recipe/common.php`, so its configuration and
deploy stages are always available. This is a curated summary of the relevant
config keys and tasks; consult the official Common Recipe docs for the full list.

## Key configuration

| Config | Meaning | Default |
| --- | --- | --- |
| `repository` | Repository to deploy | — |
| `deploy_path` | Where to deploy on the host (required) | — |
| `current_path` | Current release path | `{{deploy_path}}/current` |
| `keep_releases` | Releases to preserve in `releases/` | `10` |
| `default_timeout` | Timeout for `run()` / `runLocally()` (sec); `null` disables | `300` |
| `env` | Remote environment variables (array) | — |
| `dotenv` | Path to `.env` used per `run()` | `false` |
| `user` | User running the deploy | autodetected (git user / `whoami`) |
| `bin/php` | Path to `php` | `/usr/bin/php{{php_version}}` if host has `php_version`, else `which('php')` |
| `bin/git` | Path to `git` | `which('git')` |
| `bin/symlink` | `ln` invocation | `ln -nfs` (`--relative` when supported) |

Override env per `run()` call:

```php
run('echo $KEY', env: ['KEY' => 'over']);
```

## Deploy stages

`common` provides the building blocks the framework recipes assemble:

- **`deploy:prepare`** (group): `deploy:info` → `deploy:setup` → `deploy:lock`
  → `deploy:release` → `deploy:update_code` → `deploy:env` → `deploy:shared`
  → `deploy:writable`.
- **`deploy:publish`** (group): `deploy:symlink` → `deploy:unlock`
  → `deploy:cleanup` → `deploy:success`.
- **`deploy`** (group, in `common`): `deploy:prepare` → `deploy:publish`.
  Framework recipes redefine `deploy` to insert their own stages.

## Other tasks

- `deploy:success` — prints the success message.
- `deploy:failed` — hook invoked on deploy failure.
- `logs:app` — follows the latest application logs (uses `log_files`).

`common` itself requires the `provision` recipe and the `deploy/*` building
blocks (`check_remote`, `cleanup`, `clear_paths`, `copy_dirs`, `env`, `info`,
`lock`, `push`, `release`, `rollback`, `setup`, `shared`, `symlink`,
`update_code`, `vendors`, `writable`).
