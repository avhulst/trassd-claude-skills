# Extending DimensionContent (Page / Article / Snippet, Sulu 3.0)

Example uses `Article`; the same pattern applies to `Page` and `Snippet`.

> Note: the content/article/page packages live in separate Composer packages
> (namespaces `Sulu\Content\…`, `Sulu\Article\…`, `Sulu\Page\…`,
> `Sulu\Snippet\…`); they are not part of the `sulu/sulu` core repository.

## 1. Replace both models

Replace the content-rich entity **and** the dimension content model.

```yaml
# config/packages/sulu_article.yaml
sulu_article:
    objects:
        article:
            model: App\Entity\Article
        article_content:
            model: App\Entity\ArticleDimensionContent
```

Page / Snippet equivalents:

```yaml
# sulu_page.yaml
sulu_page:
    objects:
        page:         { model: App\Entity\Page }
        page_content: { model: App\Entity\PageDimensionContent }

# sulu_snippet.yaml
sulu_snippet:
    objects:
        snippet:         { model: App\Entity\Snippet }
        snippet_content: { model: App\Entity\SnippetDimensionContent }
```

## 2. Content-rich entity returns your dimension content (mandatory)

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Sulu\Article\Domain\Model\Article as SuluArticle;
use Sulu\Content\Domain\Model\DimensionContentInterface;

#[ORM\Entity]
#[ORM\Table(name: 'ar_articles')] // keep original table when replacing in place
class Article extends SuluArticle
{
    public function createDimensionContent(): DimensionContentInterface
    {
        return new ArticleDimensionContent($this);
    }
}
```

Original tables to keep:
`pa_pages` / `pa_page_dimension_contents`,
`ar_articles` / `ar_article_dimension_contents`,
`sn_snippets` / `sn_snippet_dimension_contents`.

## 3. Custom DimensionContent entity

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;
use Sulu\Article\Domain\Model\ArticleDimensionContent as SuluArticleDimensionContent;
use Sulu\Article\Domain\Model\ArticleInterface;

#[ORM\Entity]
#[ORM\Table(name: 'ar_article_dimension_contents')]
class ArticleDimensionContent extends SuluArticleDimensionContent
{
    #[ORM\Column(type: Types::INTEGER, options: ['default' => 0])]
    private int $status = 0;

    /** @var array<string, mixed> */
    #[ORM\Column(type: Types::JSON)]
    private array $tabData = [];

    public function __construct(ArticleInterface $article)
    {
        parent::__construct($article);
    }

    public function getStatus(): int
    {
        return $this->status;
    }

    public function setStatus(int $status): static
    {
        $this->status = $status;

        return $this;
    }

    /** @return array<string, mixed> */
    public function getTabData(): array
    {
        return $this->tabData;
    }

    /** @param array<string, mixed> $tabData */
    public function setTabData(array $tabData): static
    {
        $this->tabData = $tabData;

        return $this;
    }
}
```

## 4. ContentDataMapper (writing) — mandatory for custom fields

Write to the localized dimension content for per-locale values, to the
unlocalized one for values shared across locales.

```php
<?php

declare(strict_types=1);

namespace App\Content\DataMapper;

use App\Entity\ArticleDimensionContent;
use Sulu\Content\Application\ContentDataMapper\DataMapper\DataMapperInterface;
use Sulu\Content\Domain\Model\DimensionContentInterface;

final class ArticleCustomFieldsDataMapper implements DataMapperInterface
{
    public function map(
        DimensionContentInterface $unlocalizedDimensionContent,
        DimensionContentInterface $localizedDimensionContent,
        array $data,
    ): void {
        if (!$localizedDimensionContent instanceof ArticleDimensionContent) {
            return;
        }

        if (\array_key_exists('metadata', $data)
            && \is_array($data['metadata'])
            && \array_key_exists('status', $data['metadata'])
        ) {
            $localizedDimensionContent->setStatus((int) $data['metadata']['status']);
        }

        if (\array_key_exists('tabData', $data) && \is_array($data['tabData'])) {
            $localizedDimensionContent->setTabData($data['tabData']);
        }
    }
}
```

## 5. ContentMerger — for custom fields in merged content

```php
<?php

declare(strict_types=1);

namespace App\Content\Merger;

use App\Entity\ArticleDimensionContent;
use Sulu\Content\Application\ContentMerger\Merger\MergerInterface;

final class ArticleCustomFieldsMerger implements MergerInterface
{
    public function merge(object $targetObject, object $sourceObject): void
    {
        if (!$targetObject instanceof ArticleDimensionContent) {
            return;
        }

        if (!$sourceObject instanceof ArticleDimensionContent) {
            return;
        }

        $targetObject->setStatus($sourceObject->getStatus());
        $targetObject->setTabData(\array_merge(
            $targetObject->getTabData(),
            $sourceObject->getTabData(),
        ));
    }
}
```

## 6. ContentNormalizer — only when API output must differ

```php
<?php

declare(strict_types=1);

namespace App\Content\Normalizer;

use App\Entity\ArticleDimensionContent;
use Sulu\Content\Application\ContentNormalizer\Normalizer\NormalizerInterface;

final class ArticleCustomFieldsNormalizer implements NormalizerInterface
{
    public function enhance(object $object, array $normalizedData): array
    {
        if (!$object instanceof ArticleDimensionContent) {
            return $normalizedData;
        }

        $normalizedData['metadata'] = ['status' => $object->getStatus()];

        return $normalizedData;
    }

    public function getIgnoredAttributes(object $object): array
    {
        if (!$object instanceof ArticleDimensionContent) {
            return [];
        }

        return ['status']; // exposed under metadata/status instead
    }
}
```

## 7. Register the services

Priorities are examples; tune relative to built-in mappers/normalizers.

```yaml
services:
    App\Content\DataMapper\ArticleCustomFieldsDataMapper:
        tags:
            - { name: 'sulu_content.data_mapper', priority: 64 }

    App\Content\Normalizer\ArticleCustomFieldsNormalizer:
        tags:
            - { name: 'sulu_content.normalizer', priority: 20 }

    App\Content\Merger\ArticleCustomFieldsMerger:
        tags:
            - { name: 'sulu_content.merger', priority: 20 }
```

## Storage decision

- **content-rich entity** — global value, not locale/stage/version-bound
  (page tree info, webspace, identifiers); rare for editor fields.
- **dimension content, scalar column** — lifecycle value used in repository
  filters / Doctrine queries / list builders / sorting / indexes (status,
  priority, dates, import ids).
- **dimension content, relation** — value has own identity/lifecycle or needs
  referential integrity (m:n selections, reusable sets, own admin UI).
  Existing examples: `PageDimensionContent.navigationContexts`,
  `ArticleDimensionContent.additionalWebspaces`.
- **dimension content, JSON column** — grouped low-query settings / nested tab
  data. Never store query-critical fields in JSON.
