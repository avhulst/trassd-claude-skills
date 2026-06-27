---
name: ddev-cms-settings
description: >-
  Integrate DDEV with CMS/framework projects — auto-generated CMS settings,
  database connection wiring, and database management (import/export/snapshot).
  Triggers when configuring a CMS (Drupal, TYPO3, WordPress, Shopware, …) to run
  under DDEV or importing/exporting its database.
---

# DDEV CMS Settings & Database Management

Guidance for wiring a CMS or framework to run under DDEV: how DDEV manages
settings files automatically, the in-container database credentials your app
should use, and the commands for importing, exporting, and snapshotting the
database.

## Automatic settings management

Any CMS-specific project `type` (anything other than the generic `php` type)
gets DDEV-managed settings. For such a project, on `ddev start` DDEV will:

- Create a main settings file if none exists (e.g. Drupal's `settings.php`).
- Create a DDEV-specific specialty file with database/connection config
  (e.g. `settings.ddev.php` for Drupal, `config/system/additional.php` for
  TYPO3, `wp-config-ddev.php` for WordPress).
- Add an include of the specialty file from the main settings file when needed.

CMS types with settings management documented in DDEV: **Backdrop, Drupal,
TYPO3, WordPress.** (These are the CMSes with specifics in the DDEV CMS-settings
docs. The generic `php` type gets no settings management.)

### The `#ddev-generated` marker

Files DDEV manages contain a `#ddev-generated` comment. DDEV will keep updating
any file carrying that marker — meaning you don't have to touch it, but any edit
you make gets overwritten on the next `ddev start`.

To take ownership of a generated file: **remove the `#ddev-generated` line.**
DDEV then leaves the file entirely under your control. Check it into version
control. To hand the file back to DDEV, delete it and run `ddev start` — DDEV
recreates its own version.

### Backing off settings management entirely

Four ways to reduce or stop DDEV's CMS settings management:

1. Remove the `#ddev-generated` comment from a specific file (take ownership of
   just that file; see above).
2. Disable it project-wide: set `disable_settings_management: true` in
   `.ddev/config.yaml`, or run `ddev config --disable-settings-management`.
   DDEV uses the CMS project type but creates no settings files.
3. Switch to the generic `php` project type (`type: php` in config, or
   `ddev config --project-type=php`). No settings files are created or tweaked;
   you also lose the CMS-tuned nginx config.
4. Unset `$IS_DDEV_PROJECT`. This env var is `true` by default inside DDEV; the
   active parts of `settings.ddev.php` / TYPO3's `additional.php` are fenced
   behind it, so they don't execute outside DDEV (e.g. if a generated file
   leaks into production).

Note: DDEV creates `.ddev/.gitignore` on `ddev start` when settings management
is enabled. Do **not** commit that file — it ignores itself and DDEV's
auto-managed/temporary files, which keeps a shared `.ddev` folder portable
across DDEV versions.

## In-container database connection

DDEV creates a default database with consistent credentials. Configure your CMS
(or read in your own settings file once you've taken ownership) to connect with:

| Setting       | Value          |
| ------------- | -------------- |
| Host          | `db`           |
| Database name | `db`           |
| User          | `db`           |
| Password      | `db`           |
| Port          | `3306` (MySQL/MariaDB) / `5432` (PostgreSQL) |

The hostname `db` resolves only inside the Docker network (from the `web`
container). Supported engines include MariaDB, MySQL, and PostgreSQL. For an
on-host GUI client, run `ddev describe` for external `Host:` connection details,
or set a static `host_db_port` in `.ddev/config.yaml`; the username/password are
still `db` / `db`.

## Database management commands

```bash
# Import (formats: .sql, .sql.gz, .mysql, .mysql.gz, .tar, .tar.gz, .zip)
ddev import-db --file=dumpfile.sql.gz

# Export the default `db` database (gzipped)
ddev export-db -f mysite.sql.gz

# Direct client access to the db container
ddev mysql                                  # interactive
ddev mysql -udb -pdb                         # with db user privileges
echo "SHOW TABLES;" | ddev mysql             # piped query
ddev mysql -e 'SHOW TABLES LIKE "node%";'    # one-off query
ddev psql                                    # PostgreSQL projects
```

### Extra (named) databases

Use `--database` (alias `-d`) to target a database other than the default `db`:

```bash
ddev import-db --database=backend --file=backend.sql.gz   # creates + fills `backend`
ddev export-db --database=backend -f backend-export.sql.gz
```

### Snapshots

Snapshots capture the entire state of the db server (all databases plus the
system `mysql`/`postgres` database) as gzipped files in `.ddev/db_snapshots`.

```bash
ddev snapshot --name=before-migration        # named snapshot
ddev snapshot restore before-migration        # restore by name
ddev snapshot restore --latest                 # restore most recent
ddev snapshot restore                           # choose interactively
ddev snapshot --cleanup                         # remove snapshots to free space
```

To change database **type**: export your data, `ddev delete` the project,
change the type in config, `ddev start`, then `ddev import-db`.

See [references/cms-specifics.md](references/cms-specifics.md) for per-CMS notes
(Drupal multisite, TYPO3 base variants, WordPress docroot/environment) and
[references/db-dumps.md](references/db-dumps.md) for `mysqldump`/`pg_dump` usage.
