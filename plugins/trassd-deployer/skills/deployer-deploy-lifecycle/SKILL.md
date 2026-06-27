---
name: deployer-deploy-lifecycle
description: Configure Deployer's common-recipe deploy lifecycle — the releases/current/shared directory layout, the deploy:prepare and deploy:publish phases, shared_files & shared_dirs, writable_dirs, keep_releases retention, the atomic symlink switch, and rollbacks. Use when setting up shared or writable paths, tuning release retention, configuring the symlink switch, or wiring rollback behavior in a deploy.php recipe.
---

# Deployer: Deploy Lifecycle

Importing `recipe/common.php` gives a project the standard zero-downtime deploy flow.
Use this skill when configuring how releases, shared state, permissions, and rollbacks
behave.

```php
namespace Deployer;

import('recipe/common.php');

set('repository', 'git@github.com:acme/app.git');
host('example.org')->set('deploy_path', '~/app');
```

`deploy_path` is **required** (Deployer throws if it is unset).

## Directory layout

Under `{{deploy_path}}`, the common recipe maintains:

```
{{deploy_path}}/
├── current -> releases/N      # symlink to the live release
├── releases/                  # numbered release dirs (1, 2, 3, …)
├── shared/                    # files & dirs persisted across releases
└── .dep/                      # Deployer metadata (latest_release, releases_log)
```

Key config: `current_path` (defaults to `{{deploy_path}}/current`), `release_path`
(the release being deployed), and `release_or_current_path` (the release path during a
deploy, falling back to `current_path` otherwise — use it for paths like `dotenv`).

## The deploy phases

`deploy` is a group task composed of two phases:

- **`deploy:prepare`** — `deploy:info`, `deploy:setup`, `deploy:lock`, `deploy:release`,
  `deploy:update_code`, `deploy:env`, `deploy:shared`, `deploy:writable`. This creates the
  new numbered release dir, fetches code, and links shared/writable paths — all **without
  touching the live site**.
- **`deploy:publish`** — `deploy:symlink`, `deploy:unlock`, `deploy:cleanup`,
  `deploy:success`. This atomically switches `current` to the new release and prunes old
  ones.

Because the live `current` symlink is only moved in `deploy:publish`, a failure during
`deploy:prepare` leaves the running release untouched. Hook custom work into named tasks
(e.g. `before('deploy:symlink', 'build')`) rather than redefining `deploy`.

## Shared files and dirs

Anything that must survive across releases (env files, user uploads, logs) is stored once
in `{{deploy_path}}/shared` and symlinked into each release by `deploy:shared`:

```php
set('shared_files', ['.env']);
set('shared_dirs', ['storage', 'var/log']);
```

Both default to empty arrays. On first deploy, if the file/dir exists in the release it is
moved into `shared`; thereafter the release copy is removed and replaced with a symlink to
the shared one. Do not list a shared dir that is a parent of another shared dir —
`deploy:shared` rejects overlapping entries. Append rather than overwrite with
`add('shared_files', ['…'])`.

## Writable dirs

`deploy:writable` grants the web server write access to `writable_dirs` (paths relative to
the release):

```php
set('writable_dirs', ['var/cache', 'var/log', 'storage']);
set('writable_mode', 'acl');   // default
```

`writable_mode` is one of `acl` (default), `chown`, `chgrp`, `chmod`, `sticky`, or `skip`.
Related config: `http_user` / `http_group` (auto-detected from the process list),
`writable_use_sudo` (default `false`), `writable_recursive` (`-R`, default `false`),
`writable_chmod_mode` (default `0755`), `writable_acl_groups`, and `writable_acl_force`.
Use `acl` where the filesystem supports it; fall back to `chmod`/`chown` otherwise, and
set `writable_use_sudo` if the deploy user lacks permission.

## Release retention

`deploy:cleanup` (in `deploy:publish`) removes releases older than `keep_releases`:

```php
set('keep_releases', 5);   // default: 10
```

Keep enough releases to roll back to a known-good one. Set `cleanup_use_sudo` if old
release dirs are owned by another user. `dep releases` lists releases with their status
(current / bad / dirty).

## The atomic symlink switch

`deploy:symlink` makes the new release live. When the OS supports it (`use_atomic_symlink`,
auto-detected), it uses a single atomic `mv -T` to repoint `current` — there is no instant
where `current` is missing. Otherwise it falls back to a two-step symlink replace. Keep
`use_atomic_symlink` on its auto-detected default unless you have a specific reason to
override it.

## Rollback

`dep rollback` repoints `current` to the previous good release and marks the rolled-back
release as bad by writing a `BAD_RELEASE` file (with timestamp and user) into it; bad
releases are skipped as future rollback candidates.

```bash
dep rollback                          # roll back to the auto-chosen candidate
dep rollback -o rollback_candidate=123   # roll back to a specific release
```

As a manual fallback you can re-point the symlink yourself:
`dep run '{{bin/symlink}} releases/123 {{current_path}}'`.

## Rules of thumb

- Set `deploy_path` per host; everything else (`releases`, `current`, `shared`) is derived.
- Put env files, uploads, and logs in `shared_files`/`shared_dirs` so they persist; put
  framework cache/log dirs in `writable_dirs`.
- Tune `keep_releases` to balance disk usage against how far back you may need to roll back.
- Do production-affecting work (migrations, cache warmup) via hooks **before**
  `deploy:symlink`, so a failure aborts before the release goes live.
- Leave `use_atomic_symlink` auto-detected to preserve zero-downtime switches.
