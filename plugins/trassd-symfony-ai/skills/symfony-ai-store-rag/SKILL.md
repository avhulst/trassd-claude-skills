---
name: symfony-ai-store-rag
description: >-
  Implement retrieval-augmented generation with the Symfony AI Store component —
  vector stores (local InMemory/Cache, SQLite, MongoDB Atlas, Supabase, AWS S3
  Vectors), indexing documents (vectorize + add), and similarity search for RAG.
  Triggers when setting up a vector store, indexing documents into one, building
  a RAG pipeline, or wiring similarity search into a Symfony AI agent.
---

# Symfony AI: Store & RAG

The **Store** component (`composer require symfony/ai-store`) is a low-level
abstraction for storing and retrieving documents in a vector store. It powers
**Retrieval Augmented Generation (RAG)**: convert documents to vector
embeddings, store them, find the documents most similar to a user query, and
feed that context to the model so it answers from your data instead of only its
training set.

## Core abstractions

- **`Symfony\AI\Store\StoreInterface`** — every backend implements `add()`,
  `remove()`, `query()`, `supports()`. Backends live under
  `Symfony\AI\Store\Bridge\*` (plus `Symfony\AI\Store\InMemory\Store`). Code
  against the interface so you can swap backends.
- **`Symfony\AI\Store\ManagedStoreInterface`** — backends that need schema/index
  creation add `setup()` and `drop()`.
- **`Symfony\AI\Store\Document\TextDocument`** — raw input: `id`, `content`,
  optional `Symfony\AI\Store\Document\Metadata`. Use `Symfony\Component\Uid\Uuid`
  for ids.
- **`Symfony\AI\Store\Document\VectorDocument`** — an embedded document: `id`,
  `Symfony\AI\Platform\Vector\Vector`, `Metadata`. This is what stores hold.
- **`Symfony\AI\Store\Document\Vectorizer`** — turns documents/queries into
  vectors via a Platform + embeddings model (e.g. `text-embedding-3-small`).
- **Queries** under `Symfony\AI\Store\Query\`: `VectorQuery` (similarity),
  `TextQuery` (full-text), `HybridQuery` (combine both).

## Indexing documents

Indexing runs the pipeline **filter → transform → vectorize → store**. All
indexers implement `Symfony\AI\Store\IndexerInterface` with one `index()`
method, and share `Symfony\AI\Store\Indexer\DocumentProcessor`. Pick the one
matching where documents come from:

- **`DocumentIndexer`** — you already have `EmbeddableDocumentInterface`
  instances (e.g. `TextDocument`).
- **`SourceIndexer`** — load from a source (file/URL) via a
  `Symfony\AI\Store\Document\LoaderInterface` (`TextFileLoader`, `CsvLoader`,
  `MarkdownLoader`, `JsonFileLoader`, …).
- **`ConfiguredSourceIndexer`** — wraps a `SourceIndexer` with a default source
  defined in config, still overridable at runtime.

```php
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');
$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index(new TextDocument($id, 'This is a sample document.'));
// index() also accepts an iterable of documents.
```

For chunking large documents, transform via
`Symfony\AI\Store\Document\Transformer\TextSplitTransformer` in the processor.
See [references/indexing.md](references/indexing.md).

## Similarity search

Use the higher-level **`Symfony\AI\Store\Retriever`** (vectorizes the query
string, then queries the store) for RAG, or call `$store->query()` directly when
you need backend-specific options.

```php
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($store, $vectorizer);
foreach ($retriever->retrieve('What is the capital of France?', ['maxItems' => 5]) as $doc) {
    echo $doc->getMetadata()['source'] ?? '';
}
```

```php
use Symfony\AI\Store\Query\VectorQuery;

$results = $store->query(new VectorQuery($queryVector), ['maxItems' => 5]);
```

Query options are **backend-specific** (e.g. `maxItems`/`filter` for local &
SQLite, `limit`/`numCandidates`/`minScore` for MongoDB, `topK`/`filter` for S3
Vectors, `max_items`/`min_score` for Supabase). See per-backend notes below.

## Choosing & configuring a backend

| Backend | Package / namespace | When to use |
| --- | --- | --- |
| **InMemory** | `Symfony\AI\Store\InMemory\Store` | Testing/prototyping; data lost on process end; whole dataset in PHP memory |
| **Cache (PSR-6)** | `Symfony\AI\Store\Bridge\Cache\Store` | Local persistence via a cache adapter; still loads all data into memory |
| **SQLite** | `symfony/ai-sqlite-store`, `Bridge\Sqlite\Store` / `VecStore` | Single-file persistence, no server; `VecStore` (sqlite-vec) for native KNN beyond a few thousand docs |
| **MongoDB Atlas** | `symfony/ai-mongo-db-store`, `Bridge\MongoDb\Store` | Managed cloud vector search; needs Atlas (or atlas-local), `ext-mongodb` |
| **Supabase** | `Bridge\Supabase\Store` | pgvector via REST; requires manual SQL schema + RPC function |
| **S3 Vectors** | `symfony/ai-s3vectors-store`, `Bridge\S3Vectors\Store` | AWS-native vector storage at scale; needs AsyncAws S3Vectors client |

Local stores support distance strategies via
`Symfony\AI\Store\Distance\DistanceCalculator` /
`Symfony\AI\Store\Distance\DistanceStrategy` (COSINE_DISTANCE default). Managed
backends set the metric on their index, not in PHP. Keep **vector dimension
consistent** across all documents in an index, and match it to your embedding
model. Run `bin/console ai:store:setup <name>` / `ai:store:drop <name>` when
using the AI Bundle.

Full per-backend constructor signatures, setup, query options, and SQL/index
definitions are in [references/backends.md](references/backends.md).

## Assembling a RAG pipeline

The standard flow is **retrieve → augment prompt → generate**, typically wired
as an agent tool so the model decides when to search:

1. Create/select a store and index your documents (above).
2. Wrap a `Retriever` in
   `Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch` and register it
   in a `Symfony\AI\Agent\Toolbox\Toolbox` exposed via
   `Symfony\AI\Agent\Toolbox\AgentProcessor`.
3. Build the `Symfony\AI\Agent\Agent` with that processor as in/out processor.
4. Send a `MessageBag` whose system message instructs the agent to use
   `SimilaritySearch`; the agent calls the tool, retrieves context, and answers.

```php
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($store, $vectorizer);
$processor = new AgentProcessor(new Toolbox([new SimilaritySearch($retriever)]));
// $agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);
```

Going to production: start with InMemory, then swap in a persistent backend
without changing the rest of the pipeline. With the AI Bundle the whole pipeline
(platform, vectorizer, store, indexer, agent) is wired via YAML and populated
with `ai:store:setup` / `ai:store:index`. Full worked example in
[references/rag-pipeline.md](references/rag-pipeline.md).
