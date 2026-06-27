# Creating a reusable Mate extension

A Mate extension is a Composer package discovered through its `extra.ai-mate`
section (analogous to PHPStan extensions). A starter template exists at
`matesofmate/extension-template`.

## `extra.ai-mate` config reference

- `scan-dirs` (optional) — directories (relative to package root) scanned for
  capability classes carrying MCP attributes. Default: package root. Multiple allowed.
- `includes` (optional) — array of Symfony DI PHP config files registering
  services; support `%env()%`.
- `instructions` (optional) — path to a markdown file (conventionally
  `INSTRUCTIONS.md`, relative to package root) aggregated and provided to AI
  assistants during the MCP handshake.
- `extension` (optional, default `true`) — set `false` to exclude the package from
  discovery (apps / internal tooling that consume Mate but aren't reusable extensions).

## Capabilities with DI

Tools, Resources and Prompts support constructor DI; dependencies are autowired
and injected. Tool methods must return scalar values or arrays — not objects.

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

## Service configuration

Register service config files via `extra.ai-mate.includes`:

```json
{ "extra": { "ai-mate": { "scan-dirs": ["src"], "includes": ["config/services.php"] } } }
```

```php
// config/services.php
use App\MyApiClient;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return function (ContainerConfigurator $configurator) {
    $services = $configurator->services();
    $services->set(MyApiClient::class)
        ->arg('$apiKey', '%env(MY_API_KEY)%')
        ->arg('$baseUrl', 'https://api.example.com');
};
```

For DI issues, autowire/autoconfigure services and bind interfaces:

```php
$services->set(MyService::class)->autowire()->autoconfigure();
$services->alias(MyInterface::class, MyImplementation::class);
```

## Install, discover, enable

```terminal
$ composer require vendor/my-extension
$ vendor/bin/mate discover
```

Discovery adds the entry to the consumer's `mate/extensions.php`
(`'vendor/my-extension' => ['enabled' => true]`); set `enabled` to `false` to
disable. When the host project is already initialized, `composer install/update`
refreshes discovery automatically via the Mate Composer plugin. There is no manual
service wiring step — registration is discovery-driven.

## Effective agent instructions (`INSTRUCTIONS.md`)

Good instructions help the assistant choose your tools. Keep them concise (context
limits), map CLI commands to MCP tools, and highlight benefits:

```markdown
## My Extension

Use MCP tools instead of CLI for better results:

| Instead of...          | Use                   |
|------------------------|-----------------------|
| `my-cli command`       | `my-tool`             |
| `my-cli search "term"` | `my-search` with term |

### Benefits
- Structured output that AI can parse
- Better error handling and context
```

Verify instructions are picked up with `vendor/bin/mate debug:extensions` (look
for the `instructions` field).
