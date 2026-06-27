---
name: symfony-ai-platform
description: >-
  Use the Symfony AI Platform component to talk to model providers — platform
  bridges (OpenAI, Anthropic, Gemini, VertexAI, Voyage, Mistral, Bedrock,
  Ollama, …), model catalogs, and invoking chat/embedding models. Covers
  creating a Platform via a provider factory, the Model abstraction and
  capabilities, the invoke (request → result) flow, single- and multi-provider
  routing, and embedding models. Triggers when configuring a Symfony AI
  platform/model, choosing or extending a model catalog, or making model
  requests (chat completion, embeddings, streaming) through the Platform.
---

# Symfony AI — Platform Component

The Platform component (`symfony/ai-platform`) is a unified interface over many
model providers. You write provider-agnostic code; a provider-specific **bridge**
handles the wire protocol. Install with `composer require symfony/ai-platform`.

## Creating a Platform via a bridge factory

Never instantiate provider HTTP clients by hand. Each bridge ships a
`Factory` with two static methods:

- `Factory::createPlatform(...)` — convenience that wraps **one** provider in a
  ready-to-use `Symfony\AI\Platform\Platform`.
- `Factory::createProvider(...)` — returns a `ProviderInterface` for composing
  **multiple** providers into one `Platform` (see multi-provider below).

```php
use Symfony\AI\Platform\Bridge\OpenAi\Factory;

$platform = Factory::createPlatform($_ENV['OPENAI_API_KEY']);
```

Bridges live under `Symfony\AI\Platform\Bridge\<Provider>`. Common ones:
`OpenAi`, `Anthropic`, `Gemini`, `VertexAi`, `Voyage`, `Mistral`, `Bedrock`
(AWS), `Ollama`, `Azure`, `OpenRouter`, `Generic` (any OpenAI-compatible
endpoint), `ModelsDev`, `Cache`, `Failover`. Each factory takes provider-shaped
arguments — an API key for OpenAI/Anthropic/Voyage/Mistral; a host URL for
Ollama; location + project (or API key) for VertexAi. See
[references/bridges.md](references/bridges.md) for per-provider factory calls.

## The Model abstraction

`Symfony\AI\Platform\Model` is a model **name** + a set of **capabilities** +
default **options**. Bridges subclass it (`OpenAi\Gpt`, `Anthropic\Claude`,
`Gemini`, `Generic\CompletionsModel`, `Generic\EmbeddingsModel`, …).

- **Capabilities** are constants on `Symfony\AI\Platform\Capability`
  (`INPUT_MESSAGES`, `OUTPUT_TEXT`, `OUTPUT_STREAMING`, `TOOL_CALLING`,
  `INPUT_AUDIO`, `OUTPUT_IMAGE`, `THINKING`, …). Check support before relying on
  a feature: `$model->supports(Capability::THINKING)`.
- **Options** are model/platform parameters like `temperature` or
  `max_output_tokens`.

## Invoking the platform: request → result

`PlatformInterface::invoke(string|Model $model, array|string|object $input, array $options = []): DeferredResult`

Pass a **model name string** (catalog lookup) or a **Model instance** (bypasses
the catalog). Input is a plain string or a
`Symfony\AI\Platform\Message\MessageBag`. The third arg is options.

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$messages = new MessageBag(
    Message::forSystem('You are a helpful assistant.'),
    Message::ofUser('What is the capital of France?'),
);

$result = $platform->invoke('gpt-4o-mini', $messages, ['temperature' => 0.7]);
echo $result->asText();
```

`invoke()` returns a `DeferredResult`; read it with `asText()`, `asVectors()`,
`asBinary()`, `asObject()`, or `getResult()`. Because the platform sits on
Symfony HttpClient, results resolve lazily — issue several `invoke()` calls,
then read them, to run requests in parallel. Set `['stream' => true]` and iterate
`$result->asTextStream()` for token-by-token output.

When passing a Model instance, it MUST be a **bridge-specific subclass**
(`Gpt`, `Claude`, …), never the base `Model` — the base class has no client and
cannot be routed.

## Model catalogs

Resolving `'gpt-4o-mini'` to a `Model` is the job of the **model catalog**
(`ModelCatalogInterface`, reachable via `$platform->getModelCatalog()`). Each
bridge bundles a hand-curated catalog whose lifecycle is tied to the bridge
release — a brand-new or local model may not be present yet. Three ways to cope:

1. **Per-call Model instance** — build a bridge subclass and pass it to
   `invoke()`, skipping the catalog entirely.
2. **AI Bundle `model:` config** — declaratively merge models into a platform's
   catalog (good for local LM Studio / Ollama models).
3. **models.dev bridge** (`symfony/ai-models-dev-platform` + `symfony/models-dev`)
   — a `ModelsDev\ModelCatalog('<provider>')` sourced from the community
   registry, refreshed with `composer update symfony/models-dev` independent of
   the framework. Pair it with the matching bridge factory.

Details and examples in [references/model-catalogs.md](references/model-catalogs.md).

## Single vs. multi-provider routing

A `Platform` is a router over one or more `ProviderInterface` instances. Build it
from multiple `createProvider()` calls; the default
`CatalogBasedModelRouter` sends each invocation to the first provider whose
catalog knows the requested model.

```php
use Symfony\AI\Platform\Bridge\Anthropic\Factory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\OpenAi\Factory as OpenAiFactory;
use Symfony\AI\Platform\Platform;

$platform = new Platform([
    OpenAiFactory::createProvider(apiKey: $_ENV['OPENAI_API_KEY']),
    AnthropicFactory::createProvider(apiKey: $_ENV['ANTHROPIC_API_KEY']),
]);

$platform->invoke('gpt-4o', $messages);            // → OpenAI
$platform->invoke('claude-3-5-sonnet', $messages); // → Anthropic
```

For runtime fallback on errors (not catalog routing) wrap providers in
`Bridge\Failover\FailoverPlatform`. To cache responses, wrap a platform in
`Bridge\Cache\CachePlatform`. For tests, use `Test\InMemoryPlatform` (no HTTP).

## Embedding models

Embedding models return vectors instead of text. Invoke them the same way; read
with `asVectors()`, which yields `Symfony\AI\Platform\Vector\Vector` objects.

```php
$vectors = $platform->invoke('text-embedding-3-small', $textInput)->asVectors();
$vectors[0]->getData(); // [0.123, -0.456, ...]
```

Voyage (`Bridge\Voyage`) is embedding-focused and supports multimodal input
(text, base64 image, image URL) — wrap mixed parts in a `Content\Collection`.
VertexAi and Mistral also provide embedding models. See
[references/bridges.md](references/bridges.md) for Voyage and VertexAi specifics.
