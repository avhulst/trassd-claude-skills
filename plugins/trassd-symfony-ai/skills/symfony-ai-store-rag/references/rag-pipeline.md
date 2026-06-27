# Building a RAG pipeline

A RAG system converts documents into vector embeddings, stores them, finds the
documents most similar to the user's query, and feeds that context to the model.
Ideal for knowledge bases, product catalogs, support systems, and
domain-specific chatbots.

Prerequisites: the Platform, Agent, and Store components; an embeddings model
(e.g. `text-embedding-3-small`); a language model (e.g. `gpt-4o-mini`);
optionally a persistent vector store (or InMemory for testing).

## Step 1 — Initialize the store

Start with InMemory for development; swap for a persistent backend later without
touching the rest of the pipeline.

```php
use Symfony\AI\Store\InMemory\Store;

$store = new Store();
```

## Step 2 — Prepare documents

Each document needs a unique id, the content to embed/search, and metadata
preserved with it. See [indexing.md](indexing.md) for the document-building loop.

## Step 3 — Vectorize and index

```php
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');
$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($documents);
```

The `DocumentIndexer` transforms/filters (optional), generates embeddings, and
stores them. Use `SourceIndexer` to load from files/URLs instead.

## Step 4 — Similarity search tool

Expose semantic search to the agent as a tool so the model decides when to
search.

```php
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($store, $vectorizer);
$similaritySearch = new SimilaritySearch($retriever); // optional 2nd arg: result-header prompt
$toolbox = new Toolbox([$similaritySearch]);
$processor = new AgentProcessor($toolbox);
```

`SimilaritySearch` uses the retriever to find similar documents and returns the
most relevant ones. Customize the header:
`new SimilaritySearch($retriever, 'Here are the relevant results:')`.

## Step 5 — RAG-enabled agent

```php
use Symfony\AI\Agent\Agent;

$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [$processor], // input processors
    [$processor]  // output processors
);
```

## Step 6 — Query with context

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$messages = new MessageBag(
    Message::forSystem('Please answer all user questions only using SimilaritySearch function.'),
    Message::ofUser('Which movie fits the theme of the mafia?'),
);
$result = $agent->call($messages);
```

The agent analyzes the question, calls the similarity search tool, retrieves
relevant documents, and generates a context-grounded response.

## Going to production

Replace InMemory with a persistent backend (SQLite, MongoDB Atlas, Supabase, S3
Vectors, …) and keep the pipeline unchanged. For large documents, add
`Symfony\AI\Store\Document\Transformer\TextSplitTransformer` for chunking, and
use metadata filtering on the store query to scope results.

With the AI Bundle the whole pipeline (platform, vectorizer, store, indexer,
agent) is wired via YAML and populated with `bin/console ai:store:setup` and
`ai:store:index`.
