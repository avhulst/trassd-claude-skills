# Store backends

All stores implement `Symfony\AI\Store\StoreInterface`. Managed stores
(MongoDB, SQLite, S3 Vectors, …) also implement
`Symfony\AI\Store\ManagedStoreInterface` (`setup()` / `drop()`). Keep vector
dimensions consistent across all documents in an index, matching your embedding
model's output size.

## Local stores (InMemory & Cache)

Both load all data into PHP memory during queries — only for datasets that fit
the memory limit (testing/small sets).

```php
use Symfony\AI\Store\InMemory\Store;
use Symfony\AI\Store\Query\VectorQuery;

$store = new Store();
$store->add([$document1, $document2]);
$results = $store->query(new VectorQuery($vector));
```

```php
use Symfony\AI\Store\Bridge\Cache\Store;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$store = new Store(new FilesystemAdapter()); // persistence depends on the adapter
```

### Distance strategies

```php
use Symfony\AI\Store\Distance\DistanceCalculator;
use Symfony\AI\Store\Distance\DistanceStrategy;
use Symfony\AI\Store\InMemory\Store;

$store = new Store(new DistanceCalculator(DistanceStrategy::COSINE_DISTANCE));
```

Available: `COSINE_DISTANCE` (default), `EUCLIDEAN_DISTANCE`,
`MANHATTAN_DISTANCE`, `ANGULAR_DISTANCE`, `CHEBYSHEV_DISTANCE`.

Batch processing reduces peak memory (activated when both `batchSize` on the
calculator and `maxItems` in the query options are set):

```php
$store = new Store(new DistanceCalculator(
    strategy: DistanceStrategy::COSINE_DISTANCE,
    batchSize: 500, // default 100
));
$results = $store->query($vectorQuery, ['maxItems' => 10]);
```

### Query options (local & SQLite)

- `maxItems` (int) — limit results.
- `filter` (callable) — filter by metadata **before** distance calculation.

```php
use Symfony\AI\Store\Document\VectorDocument;

$results = $store->query($vectorQuery, [
    'maxItems' => 5,
    'filter' => fn (VectorDocument $doc) =>
        $doc->getMetadata()['price'] <= 100
        && $doc->getMetadata()['enabled'] === true,
]);
```

## SQLite

`composer require symfony/ai-sqlite-store`. Lightweight file-based persistence,
no server; FTS5 for full-text search. The plain `Store` loads vectors into PHP
memory for distance calc.

```php
use Symfony\AI\Store\Bridge\Sqlite\Store;

$pdo = new \PDO('sqlite:/path/to/vectors.db');
$pdo->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);

$store = new Store($pdo, 'my_vectors');           // or Store::fromPdo(...) / Store::fromDbal(...)
$store->setup();
```

Pass a `DistanceCalculator` as the third arg for a non-default strategy.

Text and hybrid search:

```php
use Symfony\AI\Store\Query\HybridQuery;
use Symfony\AI\Store\Query\TextQuery;

$store->query(new TextQuery('artificial intelligence'));
$store->query(new HybridQuery($vector, 'search terms', 0.5));
```

### VecStore (sqlite-vec)

Uses the `sqlite-vec` extension for native KNN in SQL — recommended beyond a few
thousand documents. The extension must be installed and loadable by your PDO
setup.

```php
use Symfony\AI\Store\Bridge\Sqlite\Distance;
use Symfony\AI\Store\Bridge\Sqlite\VecStore;

if (VecStore::isExtensionAvailable($pdo)) {
    $store = new VecStore($pdo, 'my_vectors', Distance::Cosine, 1536); // Distance::Cosine | Distance::L2
    $store->setup();
}
```

Vector dimension is fixed at table creation (default 1536). Bundle config: set
`vec: true`, `distance`, and `vector_dimension` under `ai.store.sqlite`.

## MongoDB Atlas

`composer require symfony/ai-mongo-db-store`, requires `ext-mongodb` and a
MongoDB Atlas cluster (or the `mongodb-atlas-local` Docker image). Standard
self-hosted MongoDB does not support Atlas Vector Search.

```php
use MongoDB\Client;
use Symfony\AI\Store\Bridge\MongoDb\Store;

$store = new Store(
    client: new Client('mongodb+srv://user:pass@cluster.mongodb.net'),
    databaseName: 'my_database',
    collectionName: 'documents',
    indexName: 'vector_index',
    vectorFieldName: 'vector',   // default 'vector'
    bulkWrite: false,            // batch all add/remove into one round-trip when true
    embeddingsDimension: 1536,
);

$store->setup(); // creates the collection + Atlas Vector Search index (no-op if present)
```

Documents are upserted by id. Query options:

```php
use Symfony\AI\Store\Query\VectorQuery;

$results = $store->query(new VectorQuery($queryVector), [
    'limit' => 10,           // default 5
    'numCandidates' => 500,  // default 200; higher = better recall, more latency
    'filter' => ['category' => 'example'], // pre-filter; field must be in the index
    'minScore' => 0.8,
]);
```

Similarity metric (`euclidean`, `cosine`, `dotProduct`) is set on the Atlas
index, not in PHP. The index must reach `READY` before queries return results.

## Supabase

`Symfony\AI\Store\Bridge\Supabase\Store` — pgvector through the REST API.
Requires **manual** schema setup (Supabase blocks arbitrary SQL over REST):
enable `vector` extension, create a table with `vector` + `jsonb` columns, an
index, and a `match_documents` RPC function. Example SQL:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    embedding vector(768) NOT NULL,
    metadata JSONB
);

CREATE OR REPLACE FUNCTION match_documents(
    query_embedding vector(768),
    match_count int DEFAULT 10,
    match_threshold float DEFAULT 0.0
) RETURNS TABLE (id UUID, embedding vector, metadata JSONB, score float)
LANGUAGE sql AS $$
    SELECT documents.id, documents.embedding, documents.metadata,
           1 - (documents.embedding <=> query_embedding) AS score
    FROM documents
    WHERE 1 - (documents.embedding <=> query_embedding) >= match_threshold
    ORDER BY documents.embedding <=> query_embedding ASC
    LIMIT match_count;
$$;

CREATE INDEX IF NOT EXISTS documents_embedding_idx
    ON documents USING ivfflat (embedding vector_cosine_ops);
```

Distance operators: `<=>` cosine (default), `<->` euclidean, `<#>` inner
product. Index: `ivfflat` (balanced) or `hnsw` (high-dimensional, PG14+).

```php
use Symfony\AI\Store\Bridge\Supabase\Store;
use Symfony\Component\HttpClient\HttpClient;

$store = new Store(
    HttpClient::create(),
    'https://your-project.supabase.co',
    'your-anon-key',
    'documents',      // table
    'embedding',      // vector field
    768,              // dimension
    'match_documents' // function
);

$results = $store->query(new VectorQuery($queryVector), [
    'max_items' => 10,
    'min_score' => 0.7,
]);
```

Batch up to 200 documents per insert request.

## AWS S3 Vectors

`composer require symfony/ai-s3vectors-store`. Needs an AWS account with S3
Vectors access and the AsyncAws S3Vectors client.

```php
use AsyncAws\S3Vectors\S3VectorsClient;
use Symfony\AI\Store\Bridge\S3Vectors\Store;

$store = new Store(
    client: new S3VectorsClient(['region' => 'us-east-1']),
    vectorBucketName: 'my-vector-bucket',
    indexName: 'my-index',
    filter: [], // optional default query filter
    topK: 3,    // optional default result count
);

$store->setup([
    'dimension' => 1536,
    'distanceMetric' => \AsyncAws\S3Vectors\Enum\DistanceMetric::COSINE, // COSINE | EUCLIDEAN | DOT_PRODUCT
    'dataType' => \AsyncAws\S3Vectors\Enum\DataType::FLOAT32,
    // 'encryption' => ['kmsKeyId' => '...'], 'tags' => ['env' => 'production']
]);

$results = $store->query(new VectorQuery($queryVector), [
    'topK' => 5,
    'filter' => ['category' => 'documentation'],
]);

$store->remove(['id1', 'id2']);
$store->drop(); // drops index + bucket
```

## Adding & reading VectorDocuments (managed stores)

```php
use Symfony\AI\Platform\Vector\Vector;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\VectorDocument;
use Symfony\Component\Uid\Uuid;

$store->add(new VectorDocument(
    id: Uuid::v4(),
    vector: new Vector([0.1, 0.2, 0.3 /* ... */]),
    metadata: new Metadata(['title' => 'My Document']),
));

foreach ($store->query(new VectorQuery($queryVector)) as $doc) {
    echo $doc->getId().' (score: '.$doc->getScore().')';
    echo json_encode($doc->getMetadata()->getArrayCopy());
}
```
