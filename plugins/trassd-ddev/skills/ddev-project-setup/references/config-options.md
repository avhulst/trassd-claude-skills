# DDEV `.ddev/config.yaml` reference

Distilled from the DDEV docs (`users/configuration/config.md`,
`users/project.md`, `users/usage/managing-projects.md`). Project-level options
live in `.ddev/config.yaml`; global options live in
`$HOME/.ddev/global_config.yaml`. Most settings take effect on `ddev start`.

Each option below also has a `ddev config` flag — `php_version` ↔
`--php-version`, etc. Run `ddev help config` for the complete flag list.

## Project identity and type

### `name`
URL-friendly project name. Must be unique across all projects. Defaults to the
enclosing directory name. Set via `ddev config --project-name=<name>` (the flag
is `project-name`, not `name`).

### `type`
The DDEV project type, default `php`. Allowed values include: `asterios`,
`backdrop`, `cakephp`, `codeigniter`, `craftcms`, `drupal`, `drupal6`,
`drupal7`, `drupal8`, `drupal9`, `drupal10`, `drupal11`, `drupal12`, `generic`,
`joomla`, `laravel`, `magento`, `magento2`, `php`, `shopware6`, `silverstripe`,
`symfony`, `typo3`, `wordpress`, `wp-bedrock`.

- `php` makes no CMS assumptions and works with any project.
- `drupal` means "latest stable Drupal" (currently `drupal11`).
- `generic` does nothing; normally paired with `webserver_type: generic`.

### `docroot`
Relative path to the document root containing `index.php` or `index.html`.
Auto-detected; falls back to the current directory.

## Language / runtime versions

### `php_version`
Default `8.4`. Can be `5.6` through `8.5`. Specify major.minor only (`7.3`),
never a patch version.

### `nodejs_version`
Node.js version for the web container, managed by `n`. Default is the current
LTS. A bare major (`22`) installs the newest `22.x` at image build time; `""`
uses the image's bundled version (fastest for CI); `auto`/`engine` read the
version from a project file (`.n-node-version`, `.node-version`, `.nvmrc`, or
`package.json` `engines.node`). The installed version is fixed at image build
time — change it with `ddev restart --no-cache` or `ddev utility rebuild`.

### `composer_version`
Composer version for the web container and `ddev composer`. Default `2`. Cached
at container build time; rebuild to change.

### `composer_root`
Relative path (from project root) to the directory containing `composer.json` —
where Composer commands run.

## Web server and database

### `webserver_type`
Default `nginx-fpm`. Can be `nginx-fpm`, `apache-fpm`, or `generic`. After
changing, run `ddev restart`. `generic` runs no web daemon — define your own
via `web_extra_daemons`.

### `database`
DB engine + version, default `mariadb:11.8`. Supported:
- MariaDB 5.5–10.8, 10.11, 11.4, 11.8, 12.3
- MySQL 5.5–8.0, 8.4
- Postgres 9–18

Example: `ddev config --database=mariadb:11.8`. Changing the DB version in place
is not automatic — consult DDEV's database-types docs for migration.

### `omit_containers`
Containers to skip. Per project: `db`, `ddev-ssh-agent`. Globally:
`ddev-router`, `ddev-ssh-agent`. Example: `omit_containers: [db]` for a project
that uses SQLite. Omitting `db` destabilizes database-dependent features.

## Hostnames and URLs

### `additional_hostnames`
Extra hostnames, each resolving to `<hostname>.ddev.site`. Example:
`["somename", "*.thirdname"]`. Wildcards work only with DNS resolution, internet
access, and the default `project_tld`.

### `additional_fqdns`
Extra fully-qualified domain names, e.g. `["example.com", "sub1.example.com"]`.
Adds entries to `/etc/hosts` — use with care.

### `project_tld`
Default top-level domain for project URLs. Defaults to `ddev.site`.

## Assets and settings management

### `upload_dirs`
Paths (from the docroot) to user-generated file directories targeted by
`ddev import-files`. Can sit outside the docroot but must stay within the
project (`../private`). Overrides CMS defaults, so list all desired dirs:
`upload_dirs: ["sites/default/files", "../private"]`.

### `disable_settings_management`
Default `false`. When `true`, DDEV does not create or update CMS-specific
settings files.

## Override files and behavior

### `override_config`
Default `false`. When `true` in a `config.*.yaml`, that file's settings override
rather than merge — needed for resets like `additional_hostnames: []`.

### `performance_mode`
Can be `global`, `none`, or `mutagen`. Mutagen async caching is on by default on
macOS and traditional Windows. Typically a global setting; project value wins.

## Lifecycle and listing recap

- `ddev config` — initialize `.ddev/` + `config.yaml`; auto-detects type/docroot.
  `ddev config --auto` accepts all defaults; `ddev config --update` re-detects
  docroot/type/defaults for an existing project.
- `ddev start` (alias `add`) — bring up the project's containers; applies most
  config changes. `--no-cache` rebuilds custom image layers; `--all`/`-a` starts
  every project; `-y` skips confirmation.
- `ddev restart` — restart so edited config takes effect; `--no-cache` to rebuild.
- `ddev describe [name]` (aliases `status`, `st`, `desc`) — services, URLs,
  ports, DB credentials.
- `ddev launch` — open in browser (optionally a path, e.g. `ddev launch /admin`;
  `--mailpit`/`-m` opens Mailpit).
- `ddev ssh` / `ddev exec <cmd>` — shell / run commands in the web container.
  Both default to the `web` service; use `-s <service>` (e.g. `-s db`) and
  `-u <user>` to change service/user.
- `ddev stop` (aliases `rm`, `remove`) — stop and remove containers
  (nondestructive unless `--remove-data`/`-R`); `--all`/`-a` for every project,
  `--unlist`/`-U` to drop from `ddev list`, `--stop-ssh-agent` to also stop the
  SSH agent.
- `ddev poweroff` (alias `powerdown`) — stop all projects and DDEV containers
  (equivalent to `ddev stop -a --stop-ssh-agent`).
- `ddev delete <name>` — destructive; removes the project and its database.
  `--omit-snapshot`/`-O` skips the safety snapshot, `--yes`/`-y` skips the
  prompt, `--all`/`-a` deletes every project.
- `ddev list` (aliases `l`, `ls`) / `ddev list --active-only` — registered
  projects + the shared Router; `--type <type>` filters by project type.
- `ddev` (no args) — interactive dashboard (TUI); disable with `no_tui: true`.

Inside a project the database host, name, user, and password are all `db`.
