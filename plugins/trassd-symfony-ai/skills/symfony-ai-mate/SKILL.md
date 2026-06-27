---
name: symfony-ai-mate
description: >-
  Use Mate — the Symfony AI developer tool (`symfony/ai-mate`) that runs a local
  MCP server exposing your PHP/Symfony app to AI assistants (Claude, Claude Code,
  JetBrains AI, Cursor, Codex, …). Covers integrating Mate into a project
  (install, `mate init`, discovery, `mate serve`, wiring it into each assistant)
  and writing reusable Mate extensions (the `extra.ai-mate` composer contract,
  `#[McpTool]` capabilities, DI, agent instructions, registration via discovery),
  plus key troubleshooting. Triggers when setting up Mate in a project or when
  creating a Mate extension.
---

# Symfony AI — Mate

Mate (`symfony/ai-mate`) is a **development-only** MCP (Model Context Protocol)
server that exposes tools, resources and prompts about your app to AI assistants.
It is framework-agnostic at its core, with optional Symfony/Monolog bridges. Never
deploy it to production.

Install as a dev dependency:

```terminal
$ composer require --dev symfony/ai-mate
```

## When to use Mate

- You want an AI assistant to introspect your running app (container services,
  profiler data, logs) instead of guessing from source.
- You want to ship project-specific or reusable MCP tools to teammates' assistants.
- It is a debugging/dev aid only — not a runtime/production component.

## Integrating Mate into a project

1. **Initialize** (creates the `mate/` config dir, `mate/src` for custom tools,
   `mate/AGENT_INSTRUCTIONS.md`, `mcp.json`, and `bin/codex` wrappers; also adds
   the `Mate\` PSR-4 dev autoload and `extra.ai-mate` block to `composer.json`):

   ```terminal
   $ vendor/bin/mate init
   $ composer dump-autoload
   ```

2. **Discover & serve.** `mate discover` scans installed packages for extensions,
   writes `mate/extensions.php`, and refreshes the managed instruction artifacts
   (`mate/AGENT_INSTRUCTIONS.md` and the managed section in `AGENTS.md`).
   `mate serve` starts the stdio MCP server.

   ```terminal
   $ vendor/bin/mate discover
   $ vendor/bin/mate serve
   ```

   After init, the bundled `symfony/ai-mate-composer-plugin` auto-runs
   `mate discover --composer` on every `composer install`/`update`.

3. **Wire it into your assistant.** Always use **absolute paths**. For example
   Claude Code:

   ```terminal
   $ claude mcp add mate $(pwd)/vendor/bin/mate serve --scope local
   $ claude mcp list   # expect: mate - ✓ Connected
   ```

   Claude Desktop / JetBrains AI use a `php` + `/absolute/path/.../vendor/bin/mate serve`
   stdio command. Codex does **not** read `mcp.json` — start it via the generated
   `./bin/codex` wrapper. Per-assistant config snippets are in
   [references/integration.md](references/integration.md).

The root project's own tools live in `mate/src` under the `Mate\` namespace; its
`composer.json` sets `extra.ai-mate.extension: false` so the app itself is **not**
discovered as a reusable extension. Configure settings in `mate/config.php`
(a `ContainerConfigurator` callback) and enable/disable extensions in
`mate/extensions.php`.

### Quick custom tool (root project)

Drop a class in `mate/src` and tag a method with `#[McpTool]`:

```php
// mate/MyTool.php
namespace Mate;

use Mcp\Capability\Attribute\McpTool;

class MyTool
{
    #[McpTool(name: 'my_tool', description: 'My custom tool')]
    public function execute(string $param): array
    {
        return ['result' => $param];
    }
}
```

### Built-in bridges and tools

- **Core:** `server-info` (PHP runtime details).
- **Symfony bridge** (`symfony/ai-symfony-mate-extension`): `symfony-services`
  (container introspection) plus `symfony-profiler-list` / `symfony-profiler-get`
  and `symfony-profiler://` resources when the profiler is installed.
- **Monolog bridge** (`symfony/ai-monolog-mate-extension`): `monolog-search`,
  `monolog-context-search`, `monolog-tail`, `monolog-list-files`,
  `monolog-list-channels`.

Useful commands: `mate clear-cache`, `mate debug:capabilities`,
`mate debug:extensions`, `mate mcp:tools:list|inspect|call`. See
[references/commands-and-config.md](references/commands-and-config.md) for options,
bridge config parameters, and feature-disabling via `MateHelper`.

## Creating a Mate extension

A Mate extension is a normal Composer package that declares an `extra.ai-mate`
section — the discovery contract. Minimum `composer.json`:

```json
{
    "name": "vendor/my-extension",
    "type": "library",
    "require": { "symfony/ai-mate": "^0.9" },
    "extra": {
        "ai-mate": {
            "scan-dirs": ["src"],
            "instructions": "INSTRUCTIONS.md"
        }
    }
}
```

Contract keys (all under `extra.ai-mate`, all optional except that the section
itself must exist to be discovered):

- `scan-dirs` — dirs (relative to package root) scanned for `#[McpTool]` (and
  Resource/Prompt) capability classes. Default: package root.
- `includes` — Symfony DI PHP config files registering services; support `%env()%`.
- `instructions` — markdown file (conventionally `INSTRUCTIONS.md`) aggregated and
  sent to the assistant during the MCP handshake.
- `extension` — set `false` to opt the package **out** of discovery (for apps or
  internal tooling that uses Mate but isn't a reusable extension).

Capability classes get **constructor dependency injection** via Symfony's DI
container; tool methods must return scalars or arrays (not objects):

```php
use Mcp\Capability\Attribute\McpTool;
use Psr\Log\LoggerInterface;

class MyTool
{
    public function __construct(private LoggerInterface $logger) {}

    #[McpTool(name: 'my-tool', description: 'What this tool does')]
    public function execute(string $param): string
    {
        $this->logger->info('Tool executed', ['param' => $param]);
        return 'Result: '.$param;
    }
}
```

**Registration is via discovery, not manual wiring.** A consumer installs the
package and runs `vendor/bin/mate discover` (or lets the Composer plugin run it),
which adds `'vendor/my-extension' => ['enabled' => true]` to `mate/extensions.php`.
Set `enabled` to `false` to disable. Packages with `extra.ai-mate.extension: false`
are excluded from discovery entirely.

Full extension service-config, agent-instruction guidance, and the
config-key reference are in [references/extensions.md](references/extensions.md).

## Troubleshooting essentials

- **Server won't start:** requires **PHP 8.2+**; verify `vendor/bin/mate` exists,
  run `mate serve` directly to read errors, `composer install` for missing deps,
  `chmod +x vendor/bin/mate` on permission errors.
- **Crashes on startup:** lint custom tools (`php -l mate/MyTool.php`) and check
  `mate/config.php` for syntax / circular-dependency issues.
- **Assistant not connecting:** use **absolute paths**, restart the assistant
  after config changes, and `claude mcp list` (Claude Code) to confirm status.
- **Extension not found/loaded:** confirm the `extra.ai-mate` section, re-run
  `mate discover`, check `mate/extensions.php` (`enabled: true`), confirm
  scan-dirs exist; `mate debug:extensions` shows discovered/enabled/loaded state.
- **Tools missing or failing:** verify `#[McpTool]` attributes, classes inside
  scan-dirs, restart the assistant; ensure tool returns scalar/array.
- **Debug logging:** `MATE_DEBUG=1` (stderr); `MATE_DEBUG_FILE=1` writes `dev.log`
  (customize path with `MATE_DEBUG_LOG_FILE`) — useful when run via an assistant.
- **Stale behavior:** `vendor/bin/mate clear-cache`.

More detail in [references/troubleshooting.md](references/troubleshooting.md).
