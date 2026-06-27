---
name: symfony-ai-mcp
description: >-
  Integrate the Model Context Protocol with the Symfony AI MCP Bundle —
  exposing tools, prompts, and resources via an MCP server (and, eventually,
  consuming MCP). Covers installing symfony/mcp-bundle, the mcp config (server
  name/version, STDIO/HTTP transports, session storage), declaring capabilities
  with PHP attributes, and building an MCP server end to end. Triggers when
  installing symfony/mcp-bundle, building an MCP server, wiring MCP tools,
  exposing capabilities to clients like Claude Desktop, or configuring
  config/packages/mcp.yaml.
---

# Symfony AI — Model Context Protocol (MCP Bundle)

The MCP Bundle wraps the official `mcp/sdk` PHP package to let a Symfony app act
as a **Model Context Protocol server**, exposing **tools**, **prompts**, and
**resources** to MCP clients (e.g. Claude Desktop, MCP Inspector). Acting as an
MCP **client** (consuming other servers) is configured under `servers:` but is
**not yet implemented** — treat that section as future-facing.

## Install & wire routing

```terminal
composer require symfony/mcp-bundle
```

Register the bundle's route loader so the HTTP endpoint exists:

```yaml
# config/routes.yaml
mcp:
    resource: .
    type: mcp
```

The loader registers a single endpoint (default path `/_mcp`) handling
GET/POST/DELETE/OPTIONS. You only need this route for the **HTTP** transport;
STDIO is served by the `mcp:server` console command.

## Configure the server

Configuration lives in `config/packages/mcp.yaml` under the `mcp:` key. Only
declare what you need — most keys have defaults.

```yaml
# config/packages/mcp.yaml
mcp:
    app: 'my-app'            # server name exposed to clients
    version: '1.0.0'         # server version exposed to clients
    description: 'My Symfony MCP server'

    client_transports:       # which transports to expose (both default false)
        stdio: true          # served via `bin/console mcp:server`
        http: true           # served via the /_mcp controller

    http:
        path: /_mcp          # HTTP endpoint path (default /_mcp)
```

Rules:

- **Enable at least one transport.** Both `client_transports.stdio` and
  `client_transports.http` default to `false`; nothing is reachable until one
  is `true`.
- **`app` and `version` identify the server** to the client in the MCP
  handshake. Optional metadata: `description`, `instructions` (free-text
  guidance for the LLM about when/how to use the server), `website_url`,
  `icons`, and `pagination_limit` (max items per list request, default 50).
- **Limit discovery to a folder** to keep scanning fast and predictable:
  ```yaml
  mcp:
      discovery:
          scan_dirs: ['src/Mcp']
  ```
  Without it, capabilities are auto-discovered across `src/`.

## Transports

- **STDIO** — for command-line clients. Run the server manually with
  `bin/console mcp:server` (the `mcp:server` command). Useful for debugging
  before connecting a client.
- **HTTP** — for web clients and the MCP Inspector, using the SDK's
  streamable HTTP transport (JSON-RPC 2.0 over HTTP, session management, CORS,
  MCP handshake) at the configured `http.path`.

## Declaring capabilities (PHP attributes)

Capabilities are **auto-discovered** from attributed classes — no manual
service registration. Put them under your `scan_dirs` (e.g. `App\Mcp`).

```php
namespace App\Mcp;

use Mcp\Capability\Attribute\McpTool;
use Mcp\Capability\Attribute\McpPrompt;
use Mcp\Capability\Attribute\McpResource;

class TimeCapabilities
{
    // Tool: an action the client can invoke with parameters.
    #[McpTool(name: 'current-time')]
    public function getCurrentTime(string $format = 'Y-m-d H:i:s'): string
    {
        return (new \DateTime('now', new \DateTimeZone('UTC')))->format($format);
    }

    // Prompt: system instructions the client can request (array of messages).
    #[McpPrompt(name: 'time-analysis')]
    public function timeAnalysis(): array
    {
        return [['role' => 'user', 'content' => 'You are a time management expert.']];
    }

    // Resource: static data the client can read (return uri/mimeType/text).
    #[McpResource(uri: 'time://current', name: 'current-time')]
    public function currentTimeResource(): array
    {
        return [
            'uri' => 'time://current',
            'mimeType' => 'text/plain',
            'text' => (new \DateTime('now'))->format('Y-m-d H:i:s'),
        ];
    }
}
```

Conventions:

- **Two attribute placement patterns** are supported: the **method-based**
  pattern above (multiple attributes on individual methods of one class), or the
  **invokable** pattern (one attribute on a class with `__invoke()`).
- A `#[McpResourceTemplate(uriTemplate: 'time://{var}', name: ...)]` attribute
  exists for dynamic, parameterized resources, but **resource templates are not
  yet functional** (awaiting MCP SDK handler support) — avoid relying on them.

## Build an MCP server end to end

1. `composer require symfony/mcp-bundle`.
2. Add the `type: mcp` route to `config/routes.yaml` (needed for HTTP).
3. Configure `config/packages/mcp.yaml`: set `app`/`version` and enable a
   transport under `client_transports` (and `http.path` for HTTP).
4. Create attributed capability classes under your `scan_dirs` (or `src/`).
5. Test: run `bin/console mcp:server` for STDIO, or point a client at the HTTP
   endpoint. Register it in the client (e.g. Claude Desktop):
   ```json
   {
     "mcpServers": {
       "my-app": { "command": "php /path/to/project/bin/console mcp:server" }
     }
   }
   ```
   For HTTP, use `"url": "http://localhost:8000/_mcp"` instead of `command`.

## Operations

- **Session storage (HTTP)** — `http.session.store` accepts `file` (default,
  uses `directory`, e.g. `%kernel.cache_dir%/mcp-sessions`), `memory`
  (non-persistent), `cache` (PSR-16 pool via `cache_pool`, default
  `cache.mcp.sessions`; Symfony pools are PSR-6, so wrap with `Psr16Cache`),
  or `framework` (Symfony session handler). All support `ttl` (default 3600)
  and `prefix`.
- **Logging** — MCP logs to a dedicated `mcp` Monolog channel; add it under
  `monolog.channels` to route it to its own handler.
- **Profiler** — when the Web Profiler is enabled, an MCP panel lists all
  registered tools, prompts, resources, and resource templates for debugging.
- **Events** — the SDK dispatches marker events
  (`Mcp\Event\ToolListChangedEvent`, `ResourceListChangedEvent`,
  `ResourceTemplateListChangedEvent`, `PromptListChangedEvent`) when
  capabilities are registered; listen via `#[AsEventListener]` to react (cache
  invalidation, logging, client notification).

## Pitfalls

- Forgetting the `type: mcp` route → HTTP transport returns 404.
- Leaving both transports `false` → server exposes nothing.
- Placing capability classes outside `scan_dirs` → silently not discovered.
- Treating `servers:` (client mode) or resource templates as working — both are
  not yet implemented.
