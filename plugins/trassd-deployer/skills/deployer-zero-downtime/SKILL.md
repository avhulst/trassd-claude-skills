---
name: deployer-zero-downtime
description: Keep Deployer releases zero-downtime by fixing the PHP-FPM/opcache symlink problem — point SCRIPT_FILENAME/DOCUMENT_ROOT at the resolved release path (Nginx $realpath_root, Caddy resolve_root_symlink, Apache opcache.revalidate_path) and clear opcache with the cachetool recipe instead of reloading php-fpm. Use when configuring the web server for symlinked releases or debugging stale code served after a deploy.
---

# Zero-downtime deploys: the PHP-FPM / opcache symlink problem

Deployer symlinks `current` to the latest release directory:

```
current -> releases/3/
releases/
    1/
    2/
    3/
```

## The problem

PHP opcodes get cached. If `SCRIPT_FILENAME` contains the `current` symlink,
opcache keys files by that unchanging path, so on a new deploy **nothing
updates** — the server keeps serving the old release's compiled code.

The naive fix is to reload php-fpm after each deploy, but reloading can cause
**dropped or failed requests** — so it is not zero-downtime. Do **not** reload
php-fpm. The correct fix is to make the server resolve the symlink so
`SCRIPT_FILENAME` points at the real `releases/N/` path.

Check your current configuration by printing it from a request:

```php
echo $_SERVER['SCRIPT_FILENAME'];
```

If it prints a path with `current` in it (e.g.
`/home/deployer/example.com/current/index.php`), the server is configured
incorrectly and you'll serve stale code after deploys.

> If you provision servers with Deployer's provision recipe, this is already
> configured correctly and no php-fpm reload is needed.

## Fix for Nginx

Nginx exposes `$realpath_root` (the symlink-resolved document root). Use it for
both `SCRIPT_FILENAME` and `DOCUMENT_ROOT`:

```nginx
location ~ \.php$ {
  include fastcgi_params;
  fastcgi_pass unix:/var/run/php/php-fpm.sock;
  fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
  fastcgi_param DOCUMENT_ROOT   $realpath_root;
}
```

Replace any `$document_root`-based `SCRIPT_FILENAME` / `DOCUMENT_ROOT` lines with
the `$realpath_root` versions above.

## Fix for Caddy

Add `resolve_root_symlink` to the `php_fastcgi` directive:

```
php_fastcgi * unix//run/php/php-fpm.sock {
    resolve_root_symlink
}
```

## Fix for Apache

Enable `revalidate_path` in `php.ini` so opcache validates the resolved path:

```ini
opcache.revalidate_path=1
```

## Clear opcache (and APCu) with the cachetool recipe

Even with the server configured correctly, clear the cache as part of the deploy
rather than reloading php-fpm. Use the cachetool contrib recipe:

```php
require 'contrib/cachetool.php';
```

Point it at your php-fpm endpoint (unix socket or `ip:port`):

```php
set('cachetool', '/var/run/php-fpm.sock');
// or
set('cachetool', '127.0.0.1:9000');
```

Per-host settings are supported:

```php
host('staging')
    ->set('cachetool', '127.0.0.1:9000');
host('production')
    ->set('cachetool', '/var/run/php-fpm.sock');
```

Because APCu and OPcache compile and cache files, clear them **right after the
symlink is created** for the new release:

```php
after('deploy:symlink', 'cachetool:clear:opcache');
after('deploy:symlink', 'cachetool:clear:apcu');
```

Available cachetool tasks: `cachetool:clear:opcache` (OPcode cache),
`cachetool:clear:apcu` (APCu system cache), and `cachetool:clear:stat` (file
status and realpath caches).

If the deployment user can't access the php-fpm socket, use cachetool's web
adapter, which makes a request to a temporary PHP file instead:

```php
set('cachetool_args', '--web --web-path=./public --web-url=https://{{hostname}}');
```

## Why not php-fpm:reload

The `contrib/php-fpm.php` recipe does provide a `php-fpm:reload` task, but its
own docs caution **not** to reload php-fpm: some user requests could fail or not
complete during the reload. Prefer the resolved-path server config plus a
cachetool clear above; reach for `php-fpm:reload` only if you cannot configure
the server otherwise.
