# Fast-track: from empty config to a RAG agent

This is the end-to-end getting-started flow. Each step builds on the previous
`config/packages/ai.yaml`.

## 1. Install

```terminal
$ composer require symfony/ai-bundle
```

Flex registers the bundle and creates an empty `config/packages/ai.yaml`.

## 2. Platform

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
```

## 3. Agent (uses the first platform by default)

```yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
    agent:
        default:
            model: 'gpt-4o-mini'
```

## 4. System prompt

```yaml
ai:
    agent:
        assistant:
            model: 'gpt-4o-mini'
            prompt:
                text: 'You are a concise assistant for a Symfony application.'
                include_tools: true
```

## 5. Tools (opt-in)

```yaml
ai:
    agent:
        assistant:
            model: 'gpt-4o-mini'
            tools:
                - 'Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch'
```

Built-in tools that ship as standalone packages with Flex recipes need only be
installed, e.g. `composer require symfony/ai-wikipedia-tool`.

## 6. Store, vectorizer, indexer

```yaml
ai:
    store:
        chromadb:
            knowledge_base:
                collection: 'docs'
    vectorizer:
        embeddings:
            platform: 'ai.platform.openai'
            model: 'text-embedding-3-small'
    indexer:
        docs:
            loader: 'Symfony\AI\Store\Document\Loader\TextFileLoader'
            vectorizer: 'ai.vectorizer.embeddings'
            store: 'ai.store.chromadb.knowledge_base'
```

Prepare and fill the store from the console:

```terminal
$ php bin/console ai:store:setup chromadb.knowledge_base
$ php bin/console ai:store:index docs --source=/path/to/document.txt
```

## 7. Search with the agent

`SimilaritySearch` needs a `retriever` (a vectorizer paired with a store), wired
through the `services` section:

```yaml
ai:
    retriever:
        docs:
            vectorizer: 'ai.vectorizer.embeddings'
            store: 'ai.store.chromadb.knowledge_base'
    agent:
        assistant:
            model: 'gpt-4o-mini'
            prompt:
                text: 'Answer questions using only the SimilaritySearch tool. If you cannot find relevant information, say so.'
            tools:
                - 'Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch'

services:
    Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch:
        $retriever: '@ai.retriever.docs'
```

## Complete fast-track configuration

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'

    store:
        chromadb:
            knowledge_base:
                collection: 'docs'

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

    agent:
        assistant:
            model: 'gpt-4o-mini'
            prompt:
                text: 'Answer questions using only the SimilaritySearch tool. If you cannot find relevant information, say so.'
            tools:
                - 'Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch'

services:
    Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch:
        $retriever: '@ai.retriever.docs'
```

From here, grow in any direction: more platforms/agents, multi-agent routing,
message stores and chats for conversation history, or MCP.
