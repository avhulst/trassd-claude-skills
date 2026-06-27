# Mate — troubleshooting

## Server

**Won't start** — requires PHP 8.2+ (`php --version`); confirm the binary
(`ls -la vendor/bin/mate`); run `vendor/bin/mate serve` to read errors; run
`composer install` for missing deps.

**Crashes on startup** — lint custom tools (`php -l mate/MyTool.php`); verify
config loads (`php -r "require 'vendor/autoload.php'; include 'mate/config.php';"`);
check for circular dependencies in service config.

**Permission denied** — `chmod +x vendor/bin/mate`. On Windows ensure PHP is on
PATH and run `php vendor/bin/mate serve`.

## Assistant connection

Use absolute paths, restart the assistant after config changes. Claude Code:
`claude mcp list` should show `mate - ✓ Connected`; otherwise remove and re-add.
Codex: use `./bin/codex` (it ignores `mcp.json`) and run `vendor/bin/mate discover`.

## Extensions

**Not discovered** — ensure the package has an `extra.ai-mate` section; run
`vendor/bin/mate discover`; check `mate/extensions.php`. A package with
`extra.ai-mate.extension: false` will never appear there.

**Not loading** — confirm `enabled: true` in `mate/extensions.php`; verify
scan-dirs exist and contain PHP files with MCP attributes; lint for PHP errors.

## Tools

**Not appearing** — verify `#[McpTool]` attributes; classes must live inside the
configured scan-dirs; restart the assistant; check server logs.

**Execution fails** — return scalars or arrays (objects aren't serializable);
check for exceptions; verify injected dependencies.

## Debug logging

- `MATE_DEBUG=1 vendor/bin/mate serve` — debug output to stderr (service
  registration, discovery, tool execution, internal state).
- `MATE_DEBUG_FILE=1` — write a `dev.log` in the cwd (useful behind an assistant
  where stderr is hidden); customize path with `MATE_DEBUG_LOG_FILE=/path/debug.log`.
- Combine: `MATE_DEBUG=1 MATE_DEBUG_FILE=1 vendor/bin/mate serve`.

## Other

- Stale behavior: `vendor/bin/mate clear-cache`.
- Test a tool directly with a small script: `new Mate\MyTool()->execute('test')`.
- Still stuck: collect PHP version, Mate version, error logs, repro steps and
  sanitized config; search/open issues at https://github.com/symfony/ai/issues.
