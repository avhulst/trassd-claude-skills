# Model catalogs — keeping model definitions current

A model name like `gpt-4o-mini` is resolved to a `Symfony\AI\Platform\Model`
(name + `Capability` set + default options) by a **model catalog**, an
implementation of `Symfony\AI\Platform\ModelCatalog\ModelCatalogInterface`. Each
bridge ships its own hand-curated catalog, so its coverage is **tied to the
bridge release cycle** — a freshly released or local model may not exist in the
catalog yet, and invoking it by name fails. The approaches below trade control
for convenience and can be combined.

## Approach 1 — pass a Model instance per call

The most direct escape hatch: build a fully defined, **bridge-specific** model
subclass and hand it to `invoke()` instead of a name. This bypasses the catalog
entirely, so the model need not exist in any catalog.

```php
use Symfony\AI\Platform\Bridge\OpenAi\Gpt;
use Symfony\AI\Platform\Capability;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$model = new Gpt('gpt-newest', [
    Capability::INPUT_MESSAGES,
    Capability::OUTPUT_TEXT,
    Capability::TOOL_CALLING,
], ['temperature' => 0.5]);

$result = $platform->invoke($model, new MessageBag(
    Message::ofUser('What is the Symfony framework?'),
));
```

A `string` keeps catalog-based routing; a `Model` instance bypasses it. Use this
when model definitions live in a database or behind feature flags. You must pass
a bridge subclass (`Gpt`, `Claude`, `Gemini`, …) — the base `Model` has no client
and cannot be routed.

## Approach 2 — add models via AI Bundle config

In a Symfony app using the AI Bundle, extend a platform's catalog declaratively
with the `model:` key — no per-call override, no custom catalog service. Natural
fit for local LM Studio / Ollama models or one not yet added to a bridge:

```yaml
# config/packages/ai.yaml
ai:
    platform:
        lmstudio:
            host_url: '%env(LMSTUDIO_HOST_URL)%'

    model:
        lmstudio:
            qwen3-coder-next:
                class: 'Symfony\AI\Platform\Bridge\Generic\CompletionsModel'
                capabilities:
                    - !php/const Symfony\AI\Platform\Capability::INPUT_MESSAGES
                    - !php/const Symfony\AI\Platform\Capability::OUTPUT_TEXT
                    - !php/const Symfony\AI\Platform\Capability::OUTPUT_STREAMING
                    - !php/const Symfony\AI\Platform\Capability::TOOL_CALLING
```

Models are keyed by platform name. Provide a `class` (must extend `Model`; for
generic OpenAI-compatible platforms use `Generic\CompletionsModel` for chat or
`Generic\EmbeddingsModel` for embeddings) and the `capabilities` list. Configured
models merge into the built-in catalog and take precedence over models with the
same name, so the same key also overrides an existing model's capabilities.

## Approach 3 — a continuously updated catalog (models.dev)

For broad, always-current coverage, the models.dev bridge replaces a bridge's
bundled catalog with one sourced from the community `models.dev` registry,
shipped as the standalone `symfony/models-dev` package — a daily snapshot of
providers, models, capabilities, and pricing. The **catalog lifecycle is
decoupled from the framework release cycle**: new models arrive with
`composer update symfony/models-dev`, no `symfony/ai-platform` bump or hand
editing.

```
composer require symfony/ai-models-dev-platform symfony/models-dev
```

`Bridge\ModelsDev\ModelCatalog('<provider>')` reads models.dev data for one
provider and carries the model class that bridge expects. For OpenAI-compatible
providers, pair it with the Generic bridge:

```php
use Symfony\AI\Platform\Bridge\Generic\Factory as GenericFactory;
use Symfony\AI\Platform\Bridge\ModelsDev\ModelCatalog;

$platform = GenericFactory::createPlatform(
    baseUrl: 'https://api.deepseek.com',
    apiKey: $_ENV['DEEPSEEK_API_KEY'],
    modelCatalog: new ModelCatalog('deepseek'),
);

$result = $platform->invoke('deepseek-chat', $messages);
```

For a provider with a specialized bridge, pair the catalog with that bridge —
the catalog already carries the right model class (e.g. `Claude` for Anthropic):

```php
use Symfony\AI\Platform\Bridge\Anthropic\Factory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\ModelsDev\ModelCatalog;

$platform = AnthropicFactory::createPlatform(
    apiKey: $_ENV['ANTHROPIC_API_KEY'],
    modelCatalog: new ModelCatalog('anthropic'),
);
```

Embedding models are detected automatically and wired to the matching embeddings
class; completions models carry `OUTPUT_STREAMING`, and tool-calling models are
flagged with `TOOL_CALLING`. Add or override models with the `additionalModels`
argument (merged with, and taking precedence over, the bundled data). The
`ModelsDev\ProviderRegistry` resolves API base URLs and lists providers:
`$registry->has('deepseek')`, `$registry->getProviderName('deepseek')`,
`$registry->getApiBaseUrl($id)` (null when unknown — pass the URL manually).

## Combining providers in one Platform

Each models.dev-backed bridge is a regular provider, so compose several into one
`Platform` and let it route by model name (first provider in array order whose
catalog knows the id wins):

```php
use Symfony\AI\Platform\Bridge\Anthropic\Factory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\Generic\Factory as GenericFactory;
use Symfony\AI\Platform\Bridge\ModelsDev\ModelCatalog;
use Symfony\AI\Platform\Bridge\ModelsDev\ProviderRegistry;
use Symfony\AI\Platform\Platform;

$registry = new ProviderRegistry();

$platform = new Platform([
    GenericFactory::createProvider(
        baseUrl: $registry->getApiBaseUrl('deepseek'),
        apiKey: $_ENV['DEEPSEEK_API_KEY'],
        modelCatalog: new ModelCatalog('deepseek'),
        name: 'deepseek',
    ),
    AnthropicFactory::createProvider(
        apiKey: $_ENV['ANTHROPIC_API_KEY'],
        modelCatalog: new ModelCatalog('anthropic'),
    ),
]);

$platform->invoke('deepseek-chat', $messages);     // → deepseek
$platform->invoke('claude-haiku-4-5', $messages);  // → anthropic
```

## Choosing an approach

- **One or two custom/local models** — Approach 2 (bundle config), or Approach 1
  without the bundle.
- **Model definitions from a database / feature flags / admin UI** — Approach 1,
  building the `Model` from that source.
- **Broad, always-current coverage across many providers** — Approach 3.

They compose: a models.dev-backed catalog for the long tail, plus a bundle entry
or per-call instance for the one model no registry has yet.
