# AI Bundle config reference

All configuration lives under the root `ai:` key in `config/packages/ai.yaml`.

## Supported platforms

Each platform entry registers a service `ai.platform.<name>`. API-key providers
require an `api_key`; others take provider-specific options.

```yaml
ai:
    platform:
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        gemini:
            api_key: '%env(GEMINI_API_KEY)%'
        perplexity:
            api_key: '%env(PERPLEXITY_API_KEY)%'
        elevenlabs:
            api_key: '%env(ELEVEN_LABS_API_KEY)%'
        ollama:
            endpoint: '%env(OLLAMA_HOST_URL)%'
        azure:
            # multiple deployments possible, keyed by name
            gpt_deployment:
                base_url: '%env(AZURE_OPENAI_BASEURL)%'
                deployment: '%env(AZURE_OPENAI_GPT)%'
                api_key: '%env(AZURE_OPENAI_KEY)%'
                api_version: '%env(AZURE_GPT_VERSION)%'
        bedrock:
            # multiple instances possible, e.g. per region
            default: ~
            eu:
                bedrock_runtime_client: 'async_aws.client.bedrock_runtime_eu'
        vertexai:
            location: '%env(GOOGLE_CLOUD_LOCATION)%'
            project_id: '%env(GOOGLE_CLOUD_PROJECT)%'
            api_key: '%env(GOOGLE_CLOUD_VERTEX_API_KEY)%' # optional, ADC by default
        transformersphp: ~
```

Azure and Bedrock are keyed maps (multiple named deployments/instances), so they
produce ids like `ai.platform.azure.gpt_deployment` and
`ai.platform.bedrock.default`.

## Generic and OpenResponses bridges

Configure any OpenAI-compatible service (e.g. LiteLLM) via the generic bridge:

```yaml
ai:
    platform:
        generic:
            litellm:
                base_url: '%env(LITELLM_HOST_URL)%'
                api_key: '%env(LITELLM_API_KEY)%'
                model_catalog: 'Symfony\AI\Platform\Bridge\Generic\ModelCatalog'
    agent:
        test:
            platform: 'ai.platform.generic.litellm'
            model: 'mistral-small-latest'
            tools: false
```

Services speaking the OpenAI Responses API use the `openresponses` platform with
`base_url`, `api_key`, `responses_path`, and an optional `model_catalog`.

## Cached platform decorator

Decorate a platform with a cache adapter to reduce network calls:

```yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
        cache:
            openai:
                platform: 'ai.platform.openai'
                service: 'cache.app'
    agent:
        openai:
            platform: 'ai.platform.cache.openai'
            model: 'gpt-4o-mini'
```

## Per-platform HTTP client

Every platform defaults to the `http_client` service; override per platform:

```yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
            http_client: 'app.custom_http_client'
```

## Model configuration

Append options as query params on the model name, OR use a `model.options` map —
not both:

```yaml
ai:
    agent:
        a:
            model: 'gpt-4o-mini?temperature=0.7&max_output_tokens=2000&stream=true'
        b:
            model:
                name: 'gpt-4o-mini'
                options:
                    temperature: 0.7
                    max_output_tokens: 2000
                    stream: true
```

## Adding models to a platform catalog

Register a model not in the built-in catalog under the `model` key, keyed by
platform name, giving its `class` and `capabilities`:

```yaml
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
                    - !php/const Symfony\AI\Platform\Capability::TOOL_CALLING
    agent:
        coder:
            platform: 'ai.platform.lmstudio'
            model: 'qwen3-coder-next'
```

## System prompt forms

```yaml
ai:
    agent:
        a:
            model: 'gpt-4o-mini'
            prompt: 'You are a helpful assistant.'        # simple string
        b:
            model: 'gpt-4o-mini'
            prompt:
                text: 'You are a helpful assistant.'      # text OR file (not both)
                include_tools: true                        # append tool defs to prompt
        c:
            model: 'gpt-4o-mini'
            prompt:
                file: '%kernel.project_dir%/prompts/assistant.txt'
```

Array prompts also support `enable_translation` and `translation_domain`
(require `symfony/translation`).

## Memory

```yaml
ai:
    agent:
        a:
            model: 'gpt-4o-mini'
            memory: 'You have access to user preferences and history'  # static string
        b:
            model: 'gpt-4o-mini'
            memory:
                service: 'my_memory_service'   # service implementing MemoryProviderInterface
```

If no prompt is set, memory becomes the system prompt; if both are set, memory is
prepended to the prompt.

## Multi-agent orchestration

Each `multi_agent.<name>` registers `ai.multi_agent.<name>`. `orchestrator`,
`handoffs` (at least one) and `fallback` are required; handoff keys are agent
names (auto-prefixed with `ai.agent.`):

```yaml
ai:
    multi_agent:
        support:
            orchestrator: 'orchestrator'
            handoffs:
                technical: ['bug', 'problem', 'error', 'code', 'debug']
            fallback: 'general'
```

## Message stores and chats

```yaml
ai:
    message_store:
        cache:
            youtube:
                service: 'cache.app'
                key: 'youtube'
    chat:
        youtube:
            agent: 'ai.agent.youtube'
            message_store: 'ai.message_store.cache.youtube'
```

## Console commands

| Command | Purpose |
| --- | --- |
| `ai:platform:invoke <platform> <model> "<msg>"` | Invoke a platform directly |
| `ai:agent:call <agent>` | Interactive chat with an agent |
| `ai:store:setup <store>` | Create store infrastructure |
| `ai:store:drop <store> --force` | Drop store infrastructure |
| `ai:store:index <indexer> [--source=...]` | Index documents (loader-backed indexers only) |
