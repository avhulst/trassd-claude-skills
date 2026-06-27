---
name: ddev-project-setup
description: >-
  Set up and configure a DDEV project — `ddev config`, the .ddev/config.yaml
  (project type, docroot, PHP/Node/database versions, webserver), and the
  start/stop/managing-projects lifecycle. Triggers when adding DDEV to a
  project, editing .ddev/config.yaml, or running ddev start/stop/restart.
---

# DDEV Project Setup

DDEV is a Docker-based local development environment driven by the `ddev` CLI
and a per-project `.ddev/config.yaml`. Use this skill when initializing a new
project, editing its configuration, or driving its lifecycle.

## Initializing a project with `ddev config`

The standard flow for a new project is short:

1. Clone or create the project code.
2. `cd` into the project directory and run `ddev config` to initialize DDEV.
3. Run `ddev start` to spin up the containers.
4. Run `ddev launch` to open the project in a browser.

`ddev config` creates the `.ddev/` directory and writes `.ddev/config.yaml`.
It will try to auto-detect the project type and docroot; if it guesses wrong,
pass flags or edit `.ddev/config.yaml` afterward. Most settings have an
equivalent CLI flag, so these two are the same:

```bash
ddev config --php-version 8.4
```

```yaml
# .ddev/config.yaml
php_version: "8.4"
```

Run `ddev help config` to list every available flag. A typical initialization:

```bash
mkdir -p my-site && cd my-site
ddev config --project-type=laravel --docroot=public --php-version=8.4
ddev start
ddev launch
```

After configuring, re-run `ddev describe` to confirm the detected/declared
settings. Most config changes take effect on the next `ddev start` (use
`ddev restart` to apply them to a running project — e.g. after changing
`webserver_type`).

## Key `.ddev/config.yaml` settings

These are the project-level options you set most often. See
[references/config-options.md](references/config-options.md) for the full list,
allowed values, and inline examples.

| Setting | Purpose | Default |
| -- | -- | -- |
| `name` | URL-friendly project name; must be unique | enclosing directory name |
| `type` | DDEV project type (CMS/framework preset) | `php` |
| `docroot` | Relative path to the dir containing `index.php`/`index.html` | auto-detected |
| `php_version` | PHP version, major.minor only (e.g. `8.4`) | `8.4` |
| `webserver_type` | `nginx-fpm`, `apache-fpm`, or `generic` | `nginx-fpm` |
| `database` | DB engine + version, e.g. `mariadb:11.8`, `mysql:8.0`, `postgres:16` | `mariadb:11.8` |
| `nodejs_version` | Node.js version for the web container | current LTS |
| `additional_hostnames` | Extra hostnames → `<name>.ddev.site` URLs | `[]` |
| `additional_fqdns` | Extra fully-qualified domain names | `[]` |
| `omit_containers` | Containers to skip, e.g. `[db]` | `[]` |
| `composer_root` | Path (from project root) to the dir with `composer.json` | project root |
| `upload_dirs` | User-uploaded asset dirs targeted by `ddev import-files` | preset per type |

Rules to follow:

- **`type: php` is the safe general default.** It makes no CMS assumptions and
  works with any modern PHP or static project. Use a specific type (e.g.
  `drupal11`, `wordpress`, `symfony`, `laravel`, `shopware6`, `typo3`,
  `craftcms`, `joomla`) only when you want DDEV's settings-file management for
  that platform. The `generic` type does nothing and pairs with
  `webserver_type: generic`.
- **`php_version` takes only the major.minor** (`8.4`), never a patch version.
- **`database` change is migration-sensitive** — switching versions in place
  is not automatic; consult DDEV's database-types guidance before changing it.
- **The DB connection inside the project is always `db`** for host, name, user,
  and password.
- **`nodejs_version` is fixed at image build time.** Changing it requires
  `ddev restart --no-cache` (or `ddev utility rebuild`) to rebuild the
  `ddev-webserver` image.

### Environmental override files

Override `config.yaml` with `config.*.yaml` files that merge on top of it. Teams
commonly use `config.local.yaml` (gitignored by default) for per-machine
settings — for example enabling `performance_mode: mutagen` only locally, or a
different database. Add-ons also ship their own `config.*.yaml`. Set
`override_config: true` in an override file to replace rather than merge a value
(needed for statements like `additional_hostnames: []`).

## Project lifecycle commands

| Command | Effect |
| -- | -- |
| `ddev start` | Spin up the project's containers; applies most config changes. |
| `ddev restart` | Restart containers so edited config takes effect. |
| `ddev describe [name]` | Show project details: URLs, services, ports, DB creds. |
| `ddev launch` | Open the project in a browser (e.g. `ddev launch /admin`). |
| `ddev ssh` | Open a shell inside the web container. |
| `ddev exec <cmd>` | Run a command inside the web container. |
| `ddev stop` | Stop and remove a project's containers (nondestructive). |
| `ddev stop --unlist <name>` | Stop and drop from `ddev list` until next start. |
| `ddev poweroff` | Completely stop all projects and DDEV's shared containers. |
| `ddev delete <name>` | Destructive: removes project, deletes its database. |

Notes:

- `ddev stop` (aliases `rm`, `remove`) stops and removes a project's containers
  but loses nothing unless you add `--remove-data`. The project reappears in
  `ddev list` on the next `ddev start` or `ddev config`. `ddev stop --all`
  stops every project; `--unlist`/`-U` also drops it from `ddev list`.
- `ddev poweroff` (alias `powerdown`) completely stops all projects and
  containers — it's the equivalent of `ddev stop -a --stop-ssh-agent`. Use it to
  free everything (e.g. before a Docker restart).
- `ddev delete <name>` removes all information for a project, including its
  database. A database snapshot is taken by default for recovery;
  `ddev delete --omit-snapshot` (`-O`) skips it, and `--yes`/`-y` skips the
  confirmation prompt. `ddev delete --all` deletes every project.
- After a config edit on a running project, use `ddev restart`; after changing
  `nodejs_version`, `composer_version`, or rebuilding images use
  `ddev restart --no-cache`.

## Registering and listing projects

A project is registered with DDEV when you run `ddev config`/`ddev start` in its
directory. List them all:

```bash
ddev list                 # all known projects
ddev list --active-only   # only running projects
```

`ddev list` shows each project's NAME, STATUS, LOCATION, URL, and TYPE, plus the
shared `Router`. Run `ddev` with no arguments for the interactive dashboard
(TUI). Use `ddev describe` (from the project dir, or `ddev describe <name>` from
anywhere) for per-service detail.

## Settings-file management

For CMS project types, `ddev config`/`ddev start` generate a DDEV-managed
settings file (e.g. Drupal `settings.ddev.php`, WordPress `wp-config-ddev.php`,
Craft `.env`, TYPO3 `config/system/additional.php`). DDEV-managed files carry a
`#ddev-generated` comment — leave it so DDEV keeps the file current, and put
your own overrides in a separate file that loads after it (e.g.
`settings.local.php`, `wp-config-local.php`). To opt out entirely, set
`disable_settings_management: true` (or `ddev config --disable-settings-management`).
See [references/config-options.md](references/config-options.md) for related
options.
