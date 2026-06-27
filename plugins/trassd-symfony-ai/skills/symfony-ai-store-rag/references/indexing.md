# Indexing documents

Indexing converts documents into vector embeddings and stores them. The pipeline
is **filter → transform → vectorize → store**, run by
`Symfony\AI\Store\Indexer\DocumentProcessor`. All indexers implement
`Symfony\AI\Store\IndexerInterface` (`index()`).

## Preparing documents

Each `TextDocument` carries an id, content (the text that gets embedded and
searched), and optional metadata (preserved alongside, used for filtering and
for showing source info in results).

```php
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\Component\Uid\Uuid;

$documents = [];
foreach ($movies as $movie) {
    $documents[] = new TextDocument(
        id: Uuid::v4(),
        content: 'Title: '.$movie['title'].PHP_EOL.
                 'Director: '.$movie['director'].PHP_EOL.
                 'Description: '.$movie['description'],
        metadata: new Metadata($movie),
    );
}
```

## DocumentIndexer — documents you already have

Accepts `EmbeddableDocumentInterface` instances (or iterables of them) directly.

```php
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');
$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($documents);
```

## SourceIndexer — load from a source

Loads documents from a runtime-provided source (file path, URL, …) via a
`Symfony\AI\Store\Document\LoaderInterface`.

```php
use Symfony\AI\Store\Document\Loader\TextFileLoader;
use Symfony\AI\Store\Indexer\SourceIndexer;

$loader = new TextFileLoader();
$indexer = new SourceIndexer($loader, new DocumentProcessor($vectorizer, $store));
$indexer->index('/path/to/document.txt');
// or multiple sources at once:
$indexer->index(['/path/to/doc1.txt', '/path/to/doc2.txt']);
```

Built-in loaders: `CsvLoader`, `InMemoryLoader`, `JsonFileLoader`,
`MarkdownLoader`, `RssFeedLoader`, `RstLoader`, `RstToctreeLoader`,
`TextFileLoader` (all under `Symfony\AI\Store\Document\Loader\`).

## ConfiguredSourceIndexer — config-driven default source

Wraps a `SourceIndexer` with a pre-configured default source that is still
overridable at runtime.

```php
use Symfony\AI\Store\Indexer\ConfiguredSourceIndexer;

$inner = new SourceIndexer($loader, new DocumentProcessor($vectorizer, $store));
$indexer = new ConfiguredSourceIndexer($inner, '/path/to/document.txt');
$indexer->index(); // uses the configured source
```

## Custom loader

Implement `Symfony\AI\Store\Document\LoaderInterface` (one `load()` method that
yields documents).

```php
use Symfony\AI\Store\Document\LoaderInterface;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\Component\Uid\Uuid;

class MyDocumentLoader implements LoaderInterface
{
    public function load(?string $source = null, array $options = []): iterable
    {
        $content = /* ... */;
        yield new TextDocument(Uuid::v7()->toRfc4122(), $content, new Metadata($metadata));
    }
}
```

## Chunking

For long documents, add `Symfony\AI\Store\Document\Transformer\TextSplitTransformer`
to the processor so documents are split into smaller chunks before vectorization.
