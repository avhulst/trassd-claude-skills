# Custom command annotations & environment variables

Reference detail for `ddev-custom-commands-hooks`. Distilled from DDEV's custom
commands documentation.

## All supported annotations

Annotations are `##`-prefixed comment lines in the command script header.

| Annotation | Scope | Purpose |
| --- | --- | --- |
| `Description` | all | One-line summary shown in `ddev -h`. |
| `Usage` | all | Usage string; **its value is the command name**. |
| `Example` | all | Example invocation; `\n` forces a line break. |
| `Aliases` | all | Comma-separated alternate names, e.g. `cacheclear,cache-clear,cache:clear`. |
| `Flags` | all | JSON array of flag definitions (see below). |
| `AutocompleteTerms` | all | JSON array of valid argument values for tab-completion, e.g. `["enable","disable","status"]`. |
| `ProjectTypes` | all | Comma-separated project types the command is visible for, e.g. `drupal7,drupal,backdrop`. |
| `DBTypes` | all | Comma-separated database types the command is available for, e.g. `postgres`. |
| `MutagenSync` | host/web | `true` to run Mutagen sync before and after the command (recommended if the command changes project files). |
| `CanRunGlobally` | global host only | `true` to allow running from outside a DDEV project dir. No effect if `ProjectTypes` or `DBTypes` is also set. |
| `OSTypes` | host only | Comma-separated OS list: `darwin`, `windows`, `linux`. |
| `HostBinaryExists` | host only | Only run if the given file path exists, e.g. `/Applications/PhpStorm.app`. |
| `HostWorkingDir` | container only | `true` to run from the host's current working directory inside the container. |
| `ExecRaw` | container only | `true` to pass args directly to the container as-is (so e.g. `ddev yarn --help` shows yarn's help). Recommended for container commands. |

### `Flags` definition fields

`## Flags:` takes a JSON array. Each object may use:

- `Name` — flag name on the command line.
- `Shorthand` — one-letter abbreviation.
- `Usage` — help text.
- `Type` — `bool`, `string`, `int`, or `uint` (default `bool`).
- `DefValue` — default value shown in usage.
- `NoOptDefVal` — default value when the flag is given without an option.
- `Annotations` — used by cobra Bash autocomplete code.

Examples:

```text
## Flags: [{"Name":"flag","Usage":"sets the flag option"}]
## Flags: [{"Name":"flag1","Shorthand":"f","Usage":"flag1 usage"},{"Name":"flag2","Usage":"flag2 usage"}]
```

Note: with the `Flags` annotation, only the flags you list are parsed — list
all supported flags explicitly or omit the annotation, otherwise unknown flags
raise an error. `ddev help <command>` still shows the usage help.

### Dynamic autocompletion

For dynamic (computed) completion instead of `AutocompleteTerms`, add a script
of the same name under an `autocomplete/` subdirectory, e.g. for
`$HOME/.ddev/commands/web/my-command` create
`$HOME/.ddev/commands/web/autocomplete/my-command`. It receives the current
command line as arguments and should echo valid arguments separated by line
breaks (no need to filter by the last typed argument).

## Windows notes

- Container command scripts must use **LF** line endings, not CRLF.
- DDEV needs `bash` on `PATH` (Git Bash usually satisfies this).

## Environment variables provided to command scripts

A selection of the documented `DDEV_*` variables. Avoid undocumented ones.

### Useful for host scripts

`DDEV_APPROOT` (project path on host), `DDEV_DATABASE` (`type:version`),
`DDEV_DATABASE_FAMILY` (e.g. `mysql`, `postgres`), `DDEV_DOCROOT`,
`DDEV_GID`, `DDEV_GLOBAL_DIR`, `DDEV_GOARCH`, `DDEV_GOOS`, `DDEV_HOSTNAME`,
`DDEV_HOST_DB_PORT`, `DDEV_HOST_HTTP_PORT`, `DDEV_HOST_HTTPS_PORT`,
`DDEV_HOST_MAILPIT_PORT`, `DDEV_HOST_WEBSERVER_PORT`, `DDEV_MAILPIT_HTTP_PORT`,
`DDEV_MAILPIT_HTTPS_PORT`, `DDEV_MUTAGEN_ENABLED`, `DDEV_PHP_VERSION`,
`DDEV_PRIMARY_URL`, `DDEV_PRIMARY_URL_PORT`, `DDEV_PRIMARY_URL_WITHOUT_PORT`,
`DDEV_PROJECT`, `DDEV_PROJECT_STATUS`, `DDEV_PROJECT_TYPE`,
`DDEV_ROUTER_HTTP_PORT`, `DDEV_ROUTER_HTTPS_PORT`, `DDEV_SCHEME`,
`DDEV_SITENAME`, `DDEV_TLD`, `DDEV_UID`, `DDEV_USER`, `DDEV_XHGUI_HTTP_PORT`,
`DDEV_XHGUI_HTTPS_PORT`, `DDEV_WEBSERVER_TYPE` (`nginx-fpm`, `apache-fpm`,
`generic`).

### Useful for container scripts

`DDEV_APPROOT` (project path inside web container), `DDEV_DATABASE`,
`DDEV_DATABASE_FAMILY`, `DDEV_DOCROOT`, `DDEV_FILES_DIR` (deprecated),
`DDEV_FILES_DIRS`, `DDEV_GID`, `DDEV_HOSTNAME`, `DDEV_MUTAGEN_ENABLED`,
`DDEV_PHP_VERSION`, `DDEV_PRIMARY_URL`, `DDEV_PRIMARY_URL_PORT`,
`DDEV_PRIMARY_URL_WITHOUT_PORT`, `DDEV_PROJECT`, `DDEV_PROJECT_TYPE`,
`DDEV_ROUTER_HTTP_PORT`, `DDEV_ROUTER_HTTPS_PORT`, `DDEV_SCHEME`,
`DDEV_SITENAME`, `DDEV_TLD`, `DDEV_UID`, `DDEV_USER`, `DDEV_VERSION`,
`DDEV_WEBSERVER_TYPE`, `IS_DDEV_PROJECT` (`true` when PHP runs under DDEV).
