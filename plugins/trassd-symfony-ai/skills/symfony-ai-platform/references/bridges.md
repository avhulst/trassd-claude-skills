# Platform bridges — per-provider factory calls

Every bridge lives under `Symfony\AI\Platform\Bridge\<Provider>` and exposes a
`Factory` with `createPlatform(...)` (single-provider convenience) and
`createProvider(...)` (for multi-provider `Platform` composition). The factory
arguments differ per provider; the resulting `Platform` behaves identically.

## OpenAI

```php
use Symfony\AI\Platform\Bridge\OpenAi\Factory;

$platform = Factory::createPlatform($_ENV['OPENAI_API_KEY']);

// Chat completion
$text = $platform->invoke('gpt-4o-mini', $messages)->asText();

// Embeddings
$vectors = $platform->invoke('text-embedding-3-small', 'Some text')->asVectors();
```

## Anthropic (Claude)

```php
use Symfony\AI\Platform\Bridge\Anthropic\Factory;

$platform = Factory::createPlatform($apiKey);

// Prompt caching is enabled automatically; tune retention:
// 'short' (default, 5-min), 'long' (1-hour, api.anthropic.com only), 'none'
$platform = Factory::createPlatform($apiKey, cacheRetention: 'long');
```

## Gemini (Google AI)

```php
use Symfony\AI\Platform\Bridge\Gemini\Factory;

$platform = Factory::createPlatform($apiKey);
$text = $platform->invoke('gemini-2.5-flash', $messages)->asText();
```

## Vertex AI (Google Cloud)

Vertex AI gives access to Gemini and other Google models, with several
authentication paths. Install `symfony/ai-platform` and set up Google Cloud auth.

```php
use Symfony\AI\Platform\Bridge\VertexAi\Factory;

// Application Default Credentials (gcloud auth application-default login)
// or a service account key (GOOGLE_APPLICATION_CREDENTIALS) — project-scoped endpoint
$platform = Factory::createPlatform(
    $_ENV['GOOGLE_CLOUD_LOCATION'],   // e.g. us-central1
    $_ENV['GOOGLE_CLOUD_PROJECT'],
    httpClient: $httpClient,
);

$result = $platform->invoke('gemini-2.5-flash', $messages);
echo $result->asText();
```

Authentication options:

- **ADC** — `gcloud auth application-default login`, then pass location +
  project.
- **Service account key** — set `GOOGLE_APPLICATION_CREDENTIALS` to the key
  file; same factory call as ADC.
- **API key, project-scoped** — pass location + project **and** `apiKey:`.
- **API key, global endpoint** — pass only `apiKey:` (no location/project). The
  simplest setup; does not require the `google/auth` package, but API keys only
  identify the billing project — no IAM/audit/data-residency. Use a
  project-scoped endpoint for production.

Notes:

- **Model availability varies by location.** `us-central1` has the broadest
  coverage; `global` is limited. A "Publisher Model … not found" error usually
  means the model is not in your region — switch location or model.
- Token usage is on the result metadata:
  `$result->getMetadata()->get('token_usage')` returns a
  `Symfony\AI\Platform\TokenUsage\TokenUsage` with `getPromptTokens()`,
  `getCompletionTokens()`, `getTotalTokens()`.
- Vertex AI also offers server-side tools (URL Context, Google Search grounding,
  Code Execution) — see the Vertex AI server-tools docs.

## Voyage AI (embeddings)

Voyage is embedding-focused: text embedding and multimodal embedding. Requires
an API key from the Voyage dashboard.

```php
use Symfony\AI\Platform\Bridge\Voyage\Factory;

$platform = Factory::createPlatform($_ENV['VOYAGE_API_KEY'], $httpClient);

// Text embedding
$vectors = $platform->invoke('voyage-3', 'Once upon a time...')->asVectors();
```

Multimodal embedding accepts text, base64 image data, and image URLs, and allows
multiple data types per vector — wrap them in a `Content\Collection`:

```php
use Symfony\AI\Platform\Bridge\Voyage\Factory;
use Symfony\AI\Platform\Message\Content\Collection;
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\Content\Text;

$platform = Factory::createPlatform($_ENV['VOYAGE_API_KEY'], $httpClient);

$result = $platform->invoke(
    'voyage-multimodal-3',
    new ImageUrl('https://example.com/image1.jpg'),
    new Collection(new Text('Hello, world!'), new ImageUrl('https://example.com/image2.jpg')),
);

$vectors = $result->asVectors();
```

## Generic (OpenAI-compatible endpoints)

For platforms that mimic OpenAI's API (LiteLLM, OpenRouter, DeepSeek, Groq, …)
use the Generic bridge with an explicit base URL, key, HTTP client, and a model
catalog:

```php
use Symfony\AI\Platform\Bridge\Generic\Factory;

$platform = Factory::createPlatform('https://api.example.com', 'sk-xxxx', $httpClient, $modelCatalog);
$result = $platform->invoke('model-name', $messages);
```

The catalog must use `Bridge\Generic\CompletionsModel` (chat) or
`Bridge\Generic\EmbeddingsModel` (embeddings). Pairing the Generic bridge with a
`ModelsDev\ModelCatalog` avoids curating that catalog by hand — see
[model-catalogs.md](model-catalogs.md).

## Other bridges

Available bridges include (under `Symfony\AI\Platform\Bridge\`): `Mistral`,
`Bedrock` (AWS), `Ollama` (local, host URL), `LmStudio`, `Azure`, `OpenRouter`,
`Cohere`, `Perplexity`, `Replicate`, `HuggingFace`, `Cerebras`, `ElevenLabs`,
`Cartesia`, `Decart`, `DeepSeek`. Wrapper bridges `Cache` (`CachePlatform`) and
`Failover` (`FailoverPlatform`) decorate any underlying platform.
