---
name: symfony-ai-bundle-setup
description: >-
  Set up and configure the Symfony AI Bundle — installing the bundle, wiring
  platforms, agents, stores and tools through configuration, and the fast-track
  getting-started flow. Triggers when installing symfony/ai-bundle or editing
  config/packages/ai.yaml.
---

# Symfony AI Bundle Setup

The AI Bundle is the Symfony integration that wires the Platform, Agent, Store
and Chat components together. Instead of building services by hand, you declare
**platforms, agents, tools, vectorizers, indexers, retrievers and stores** in
`config/packages/ai.yaml`; the bundle registers each one as a container service.

## Install

```terminal
$ composer require symfony/ai-bundle
```

With Symfony Flex the bundle is auto-registered and an empty
`config/packages/ai.yaml` is created. You also need an API key for at least one
provider (e.g. `OPENAI_API_KEY`).

## The `ai.yaml` config tree

Everything lives under the root `ai:` key. The top-level sections are:
`platform`, `agent`, `store`, `vectorizer`, `indexer`, `retriever`,
`message_store`, `chat`, `multi_agent`, and `model`. Each becomes one or more
named services.

### Platforms — the connection to a provider

Define each provider under `platform`; every entry registers a service named
`ai.platform.<name>`. API-key platforms require `api_key`:

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
```

This registers `ai.platform.openai` and `ai.platform.anthropic`. Many providers
are supported (Azure, Bedrock, Gemini, VertexAI, Ollama, Perplexity, ElevenLabs,
and more), plus a generic OpenAI-compatible bridge and cached/decorated
platforms. See [references/config-reference.md](references/config-reference.md)
for the provider catalog and decorator forms.

### Agents — platform + model (+ prompt + tools)

An agent combines a platform, a model, and optionally a prompt and tools. The
simplest agent only needs a model and uses the **first configured platform** by
default:

```yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
    agent:
        default:
            model: 'gpt-4o-mini'
```

With multiple agents, point each at its platform explicitly via the
`ai.platform.<name>` service id:

```yaml
ai:
    agent:
        assistant:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
        researcher:
            platform: 'ai.platform.anthropic'
            model: 'claude-3-7-sonnet-latest'
```

Each agent registers a service `ai.agent.<name>`.

### Stores — vector databases

Stores are grouped by type, then by name, producing service ids
`ai.store.<type>.<name>`:

```yaml
ai:
    store:
        chromadb:
            knowledge_base:
                collection: 'docs'
```

Run `php bin/console ai:store:setup chromadb.knowledge_base` to prepare a store's
infrastructure (only for stores implementing `ManagedStoreInterface`).

### Vectorizers, indexers, retrievers

A `vectorizer` turns text into embeddings, an `indexer` fills a store, and a
`retriever` searches it. They reference platforms and stores by service id and
register `ai.vectorizer.<name>`, `ai.indexer.<name>`, `ai.retriever.<name>`:

```yaml
ai:
    vectorizer:
        embeddings:
            platform: 'ai.platform.openai'
            model: 'text-embedding-3-small'
    indexer:
        docs:
            loader: 'Symfony\AI\Store\Document\Loader\TextFileLoader'
            vectorizer: 'ai.vectorizer.embeddings'
            store: 'ai.store.chromadb.knowledge_base'
    retriever:
        docs:
            vectorizer: 'ai.vectorizer.embeddings'
            store: 'ai.store.chromadb.knowledge_base'
```

An indexer **without** a `loader` becomes a `DocumentIndexer` (feed documents in
PHP); **with** a `loader` it is a `SourceIndexer` usable from
`ai:store:index <indexer>`.

## Referencing configured services

Anything you name in `ai.yaml` is a service. Reference patterns:

- **Platforms:** `ai.platform.<name>` (e.g. `ai.platform.openai`).
- **Agents:** `ai.agent.<name>` (e.g. `ai.agent.researcher`).
- **Stores:** `ai.store.<type>.<name>` (e.g. `ai.store.chromadb.knowledge_base`).
- **Vectorizers / indexers / retrievers:** `ai.vectorizer.<name>`,
  `ai.indexer.<name>`, `ai.retriever.<name>`.

With a single agent, inject `Symfony\AI\Agent\AgentInterface` directly. With
several, target a specific one via `#[Autowire(service: 'ai.agent.<name>')]`.
See [references/usage-and-services.md](references/usage-and-services.md).

## Tools are opt-in

Services carrying `#[AsTool]` are auto-registered (tagged `ai.tool`), but **each
agent must opt in**. List tools explicitly, or set `tools: true` to inject all
registered tools. Omitting `tools` (or `false`/`null`/empty list) leaves the
agent with no tools:

```yaml
ai:
    agent:
        assistant:
            model: 'gpt-4o-mini'
            tools:
                - 'Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch'
```

## Fast-track getting-started flow

The canonical path from empty config to a RAG-capable agent:

1. **Install** `symfony/ai-bundle`.
2. **Configure a platform** under `platform` (`ai.platform.<name>`).
3. **Configure an agent** with a `model` (uses the first platform by default).
4. **Add a system prompt** — a plain string, or the array form (`text`/`file`,
   `include_tools`, translation options).
5. **Give the agent tools** — opt in via the `tools` list or `tools: true`.
6. **Configure a store, vectorizer and indexer**, then
   `ai:store:setup` and `ai:store:index` to fill the store.
7. **Search with the agent** via the `SimilaritySearch` tool backed by a
   `retriever`.

A complete fast-track `ai.yaml` and the wiring of `SimilaritySearch` to a
retriever are in
[references/fast-track.md](references/fast-track.md).

## Verifying setup from the console

- `php bin/console ai:platform:invoke <platform> <model> "<message>"`
- `php bin/console ai:agent:call <agent>` — interactive chat with an agent
- `php bin/console ai:store:setup <store>` / `ai:store:drop <store> --force`
- `php bin/console ai:store:index <indexer> [--source=...]`

## Key rules

- **One root `ai:` key.** Each section maps to named services with predictable
  ids (`ai.<section>.<name>`, stores use `ai.store.<type>.<name>`).
- **Model options:** either query-string syntax in the model name
  (`gpt-4o-mini?temperature=0.7`) **or** a `model.options` map — never both.
- **First platform wins** when an agent omits `platform`.
- **Tools are never automatic** — opt in per agent.
