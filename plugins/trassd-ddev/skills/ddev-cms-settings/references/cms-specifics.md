# CMS-specific settings notes

Per-CMS details for the settings DDEV manages. Distilled from DDEV's
CMS-settings documentation.

## Drupal

- DDEV creates `sites/default/settings.ddev.php` and adds an include for it in
  `sites/default/settings.php`. Guards keep `settings.ddev.php` inactive when
  the project is not running under DDEV; it is gitignored and should not be
  committed.
- **MySQL/MariaDB:** Drupal 9.5+ needs
  `SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED`; DDEV applies this on
  `ddev start`.
- **PostgreSQL:** Drupal needs the `pg_trm` extension; DDEV creates it on
  `ddev start`.
- **Twig debugging:** point Drupal at `development.services.yml` instead of
  `services.yml` by adding to `settings.php` / `settings.local.php`:

  ```php
  $settings['container_yamls'][] = DRUPAL_ROOT . '/sites/development.services.yml';
  ```

- **Drush + Xdebug:** Drush 13+ disables Xdebug even when it's enabled in DDEV.
  Run a single command with `ddev drush --xdebug`, or set
  `DRUSH_ALLOW_XDEBUG=1` in the container environment to allow it for every
  Drush call.

### Multisite

For each non-default site, fence DDEV behaviour behind `IS_DDEV_PROJECT` and
include the DDEV credentials, then re-point the per-site database name (the
generated `settings.ddev.php` resets the active DB to `db`):

```php
// site/{site_name}/settings.php
elseif (getenv('IS_DDEV_PROJECT') == 'true') {
  include $app_root . '/' . $site_path . '/settings.databases.ddev.inc';
}
```

```php
// site/{site_name}/settings.databases.ddev.inc
require $app_root . '/sites/default/settings.ddev.php';
$databases['default']['default']['database'] = 'site_name';
```

If you use Drush site aliases, keep the Drush shell PID alive for the life of
the container so `drush site:set @alias` persists:

```yaml
# .ddev/config.yaml
web_environment:
    - DRUSH_SHELL_PID=PERMANENT
```

## TYPO3

- On `ddev start`, DDEV creates `config/system/additional.php` containing the
  database configuration.
- **Base variant (TYPO3 9.5+):** each site needs a Site Configuration. To browse
  locally, define an application context and a matching base variant. Set the
  context via DDEV:

  ```yaml
  # .ddev/config.yaml
  web_environment:
      - TYPO3_CONTEXT=Development/DDEV
  ```

  Then add a base variant to the Site Configuration:

  ```yaml
  baseVariants:
    -
      base: 'https://example.com.ddev.site/'
      condition: 'applicationContext == "Development/DDEV"'
  ```

## WordPress

- DDEV manages `wp-config-ddev.php` (carries the `#ddev-generated` marker).
- **Non-root docroot + WP-CLI:** DDEV v1.24.5+ auto-adds `--path=$DDEV_DOCROOT`
  to `ddev wp`. For older versions, or to be explicit, create `wp-cli.yml` in
  the project root:

  ```yaml
  path: public
  ```

- **Environment type:** `wp_get_environment_type()` returns `production` by
  default. Override it either by setting an env var:

  ```bash
  ddev config --web-environment-add="WP_ENVIRONMENT_TYPE=local"
  ```

  or by taking ownership of `wp-config-ddev.php` (remove its `#ddev-generated`
  line) and defining the constant:

  ```php
  defined( 'WP_ENVIRONMENT_TYPE' ) || define( 'WP_ENVIRONMENT_TYPE', 'local' );
  ```

## Backdrop

- Backdrop stores both active and staging configuration in the filesystem by
  default. When you refresh the database (from live or while reloading locally),
  also refresh the active configuration directory so the DB structure stays in
  line with the configuration.
- Backdrop 1.28.0+ can store active configuration in the database, which bundles
  config with the database — refreshing the DB then also refreshes config. This
  pairs well with DDEV snapshots.
