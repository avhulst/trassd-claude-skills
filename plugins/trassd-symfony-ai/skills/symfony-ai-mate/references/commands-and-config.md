# Mate — commands & project configuration

## Commands

- `mate init` — create the `mate/` dir and config; wire `composer.json`.
- `mate discover` — scan vendor for `extra.ai-mate` packages, (re)generate
  `mate/extensions.php` (preserving existing enabled/disabled states, new ones
  default enabled), and refresh instruction artifacts.
- `mate serve` — start the stdio MCP server.
- `mate clear-cache` — clear the MCP server cache.
- `mate debug:capabilities` — list discovered capabilities grouped by extension.
  Options: `--format=text|json|toon` (`toon` needs `helgesverre/toon`),
  `--extension=<pkg>` (use `_custom` for root project), `--type=tool|resource|prompt|template`.
- `mate debug:extensions` — show discovered vs enabled vs loaded extensions, scan
  dirs and includes. Options: `--format=...`, `--show-all`. Status: `[enabled]`,
  `[loaded]`, `[not loaded]`.
- `mate mcp:tools:list` — compact tool overview. Options: `--filter=<pattern>`
  (wildcards), `--extension=<pkg>`, `--format=table|json|toon`.
- `mate mcp:tools:inspect <tool-name>` — full JSON schema of a tool.
- `mate mcp:tools:call <tool-name> '<json-input>'` — execute a tool from the CLI;
  `--format=pretty|json|toon`. E.g. `mate mcp:tools:call monolog-search '{"term":"error","level":"error"}'`.

## `mate/config.php` — settings & services

A callback receiving a `ContainerConfigurator`:

```php
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (ContainerConfigurator $container): void {
    $container->parameters()
        // ->set('mate.cache_dir', sys_get_temp_dir().'/mate')
        // ->set('mate.env_file', ['.env'])
    ;
    $container->services()
        // register custom services here
    ;
};
```

Reference env vars with `%env(VAR_NAME)%` in service config.

### Disabling specific features

```php
use Symfony\AI\Mate\Container\MateHelper;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (ContainerConfigurator $container): void {
    MateHelper::disableFeatures($container, [
        'symfony/ai-mate' => ['server-info'],
    ]);
};
```

## `mate/extensions.php` — enable/disable extensions

Managed by `mate discover`; editable by hand:

```php
return [
    'vendor/package-name'    => ['enabled' => true],
    'vendor/another-package' => ['enabled' => false],
];
```

## Bridge configuration parameters

Symfony bridge:

```php
$container->parameters()
    ->set('ai_mate_symfony.cache_dir', '%mate.root_dir%/var/cache')
    ->set('ai_mate_symfony.profiler_dir', '%mate.root_dir%/var/cache/dev/profiler');
```

Multi-kernel profiler dirs (profiles gain a `context` field):

```php
$container->parameters()
    ->set('ai_mate_symfony.profiler_dir', [
        'website' => '%mate.root_dir%/var/cache/website/dev/profiler',
        'admin'   => '%mate.root_dir%/var/cache/admin/dev/profiler',
    ]);
```

Profiler data redacts cookies, session, auth headers and sensitive env vars.
Custom collector formatters implement `CollectorFormatterInterface` and are
registered via the DI tag `ai_mate_symfony.profiler_collector_formatter`.

Monolog bridge:

```php
$container->parameters()
    ->set('ai_mate_monolog.log_dir', '%mate.root_dir%/var/log');
```
