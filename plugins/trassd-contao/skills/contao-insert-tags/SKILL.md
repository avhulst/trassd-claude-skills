---
name: contao-insert-tags
description: Create and use insert tags — registering an insert tag with the #[AsInsertTag] attribute, flags, and (e)safe HTML/text handling. Triggers when adding a custom insert tag, a block insert tag, or an insert tag flag, or when working with {{...}} insert tags in templates.
---

# Contao Insert Tags

Insert tags replace specific tokens in templates and database fields with
content. They follow the format `{{TAG_NAME}}`, optionally with parameters:
`{{TAG_NAME::PARAMETERS}}`. Custom insert tags, block insert tags, and flags
are registered with PHP attributes (Contao 5.2+).

## Registering a custom insert tag

Use `#[AsInsertTag]` on a class. The name must be lowercase. Implement an
interface so the resolver receives an `InsertTag` object and returns an
`InsertTagResult`. Pick the interface by whether you want nested tags
pre-resolved:

- `InsertTagResolverNestedResolvedInterface` → receives a `ResolvedInsertTag`
  (all nested tags already replaced; parameters are plain strings).
- `InsertTagResolverNestedParsedInterface` → receives a `ParsedInsertTag`
  (nested tags kept unreplaced; parameters can be `ParsedSequence` objects).

```php
use Contao\CoreBundle\DependencyInjection\Attribute\AsInsertTag;
use Contao\CoreBundle\InsertTag\Exception\InvalidInsertTagException;
use Contao\CoreBundle\InsertTag\InsertTagResult;
use Contao\CoreBundle\InsertTag\OutputType;
use Contao\CoreBundle\InsertTag\ResolvedInsertTag;
use Contao\CoreBundle\InsertTag\Resolver\InsertTagResolverNestedResolvedInterface;

#[AsInsertTag('rot13')]
class Rot13InsertTag implements InsertTagResolverNestedResolvedInterface
{
    public function __invoke(ResolvedInsertTag $insertTag): InsertTagResult
    {
        $parameter = $insertTag->getParameters()->get(0);

        if (null === $parameter) {
            throw new InvalidInsertTagException('Missing parameters for insert tag.');
        }

        return new InsertTagResult(str_rot13($parameter), OutputType::text);
    }
}
```

`{{rot13::Contao}}` now renders `Pbagnb`.

The attribute (and the equivalent service tag `contao.insert_tag`) accepts:

| Option | Meaning |
|---|---|
| `name` | Lowercase insert tag name. |
| `resolveNestedTags` | `true` resolves nested tags before the method runs; `false` keeps them unreplaced. |
| `priority` | Highest priority wins when several tags share a name. |
| `method` | Defaults to `__invoke` or the annotated method; otherwise set a method name. |

(`asFragment` is deprecated since Contao 5.3.)

## Safe output: InsertTagResult and OutputType

`InsertTagResult` carries the value plus an `OutputType` so Contao knows whether
the value is already safe HTML or untrusted text that must be escaped. This is
what makes the difference between the `{{...}}` (raw) and `{{e...}}` (escaped)
usage safe.

- `OutputType::text` — treated as plain text and escaped on output. Use this for
  anything you cannot guarantee is safe HTML (e.g. user-derived strings,
  `str_rot13` output).
- `OutputType::html` — treated as trusted markup and emitted as-is.

`OutputType` also defines `js`, `css`, and `url` for context-aware escaping.
`InsertTagResult` is immutable; use the `with*` helpers (`withValue`,
`withOutputType`, `withExpiresAt`, `withCacheTags`) to derive a new instance.

## Block insert tags

A block insert tag wraps content between an opening and a closing tag, e.g.
`{{ifmembergroup::2}}…{{endifmembergroup}}`. Register it with
`#[AsBlockInsertTag]` and declare the `endTag` (lowercase). Implement
`BlockInsertTagResolverNestedResolvedInterface` (or the `…NestedParsedInterface`
variant). The resolver additionally receives the wrapped content as a
`ParsedSequence` and must return a `ParsedSequence` — return an empty
`new ParsedSequence([])` to output nothing.

See [references/block-insert-tags.md](references/block-insert-tags.md) for a full
member-group example.

## Parameters and named attributes

Parameters arrive via `$insertTag->getParameters()`. They can be positional or
key/value (`key=value`), repeated, and read as scalars:

```php
// {{tag::key=value}}
$p->get('key');        // "value"
$p->get(0);            // "key=value"
// {{tag::a=1::a=2}}
$p->all('a');          // ["1", "2"]
// {{tag::int=1::float=1.2}}
$p->getScalar('int');  // 1
$p->getScalar('float');// 1.2
```

With a `ResolvedInsertTag`, `{{tag::foo{{nested}}bar}}` yields the fully
resolved string from `get(0)`. With a `ParsedInsertTag`, `get(0)` returns a
`ParsedSequence` whose parts are strings and unresolved `ParsedInsertTag`s.

## Insert tag flags

Flags transform a tag's output and are appended with `|`, e.g.
`{{label::MSC:reset|rot13}}`. Register a custom flag with `#[AsInsertTagFlag]`
(lowercase name, same `name`/`priority`/`method` options) implementing
`InsertTagFlagInterface`. The resolver receives the `InsertTagFlag` and the
current `InsertTagResult`, and returns a (modified) `InsertTagResult`:

```php
use Contao\CoreBundle\DependencyInjection\Attribute\AsInsertTagFlag;
use Contao\CoreBundle\InsertTag\Flag\InsertTagFlagInterface;
use Contao\CoreBundle\InsertTag\InsertTagFlag;
use Contao\CoreBundle\InsertTag\InsertTagResult;
use Contao\CoreBundle\InsertTag\OutputType;

#[AsInsertTagFlag('rot13')]
class Rot13InsertTagFlag implements InsertTagFlagInterface
{
    public function __invoke(InsertTagFlag $flag, InsertTagResult $result): InsertTagResult
    {
        return $result
            ->withValue(str_rot13($result->getValue()))
            ->withOutputType(OutputType::text); // unsafe HTML after transform → text
    }
}
```

## Caching

Replaced insert tags are stored in the public cache by default. A tag whose name
starts with `cache_`, or one carrying the `uncached` flag, becomes a private ESI
response and is not cached publicly. A handful of core tags (`date`, `ua`,
`post`, `back`, `referer`, `request_token`) are never publicly cached. To keep a
custom tag out of the public cache, add the flag at the call site:
`{{rot13::Payload|uncached}}`.

## Legacy

Before Contao 5.2, custom insert tags were implemented via the
`replaceInsertTags` hook (`#[AsHook('replaceInsertTags')]`). Prefer the
attributes above on 5.2+.
