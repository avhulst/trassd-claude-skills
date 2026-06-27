---
name: contao-caching
description: >-
  Apply caching in a Contao extension — HTTP caching (shared max-age / cache
  headers via the response context), object/result caching, and cache
  invalidation/tagging. Triggers when configuring caching for a page or
  controller, or invalidating cache in a Contao bundle.
---

# Contao caching

Contao relies heavily on **HTTP caching** and ships the Symfony Reverse Proxy
(`HttpCache`) in every Managed Edition, so every setup has a **Shared Cache**
(reverse proxy) in front of it. Get your cache headers right or responses
(e.g. content elements) will be cached when they shouldn't be. The caching
framework is built on [FOSHttpCacheBundle][fos] with the
[toflar/psr6-symfony-http-cache-store][store] backend, which adds **cache
invalidation by tags**.

## Private vs. shared caches

- **Private cache** — for a single user; never stored by a shared cache.
- **Shared cache** — reverse proxy, shared across users. Cache *invalidation*
  works only here.

Control which one may store the response via `Cache-Control`:

```php
use Symfony\Component\HttpFoundation\Response;

$response->headers->set('Cache-Control', 'private'); // single user only
$response->headers->set('Cache-Control', 'public');  // shared cache allowed
```

## Caching methods

1. **Expiration** — how long an entry lives. Use Symfony's `Cache-Control`
   abstraction rather than writing the header by hand:

   ```php
   $response->headers->addCacheControlDirective('public');
   $response->headers->addCacheControlDirective('s-maxage', 60);  // shared cache: 60s
   $response->headers->addCacheControlDirective('max-age', 600);  // private cache: 600s
   ```

   The convenience equivalents are `$response->setPublic()`,
   `$response->setMaxAge(...)` and `$response->setSharedMaxAge(...)`.

2. **Validation** — when you cannot fix a lifetime. Send `Last-Modified` or
   `ETag`; the client revalidates with `If-Modified-Since` / `If-None-Match`
   and the server may answer `304 Not Modified`.

3. **Invalidation** — "cache forever (max 1 year) until I purge it." Only for
   shared caches. Tag responses, then purge by tag. See below.

## Cache tags and invalidation

Tag a response, then later purge everything carrying that tag. Example: tag
every response that shows news ID 42 with `news-42`; when the entry changes,
purge `news-42` and all those pages are invalidated.

- **Add tags** — inject `fos_http_cache.http.symfony_response_tagger` and call
  `addTags([...])`. Tags collected during the request are attached to the
  response on `kernel.response`.
- **Invalidate tags** — inject `fos_http_cache.cache_manager`
  (`FOS\HttpCacheBundle\CacheManager`) and call `invalidateTags([...])`.
  Invalidation requests run on `kernel.terminate`.

For **fragment controllers** (content elements, front end modules) extend
`Contao\CoreBundle\Controller\AbstractFragmentController` and use its
`tagResponse([...])` convenience method instead of the raw tagger.

See [references/tagging.md](references/tagging.md) for full controller and
listener examples.

### Automatic tagging in the back end

When a back end record is created, updated or deleted, Contao **automatically**
invalidates a conventional set of tags — you do not need DCA callbacks:

- `contao.db.<table>.<id>` — the record itself
- `contao.db.<table>` — only if the DCA has no parent table
- parent tables (recursively up): `contao.db.<parent>.<pid>`, and
  `contao.db.<parent>` for the topmost parent only
- child tables (recursively down): `contao.db.<child>.<cid>`

If your front end responses follow this `contao.db.<table>.<id>` convention,
back end edits invalidate them for free — no callbacks needed. To add extra
tags during invalidation, use the [`oninvalidate_cache_tags` callback][cb].

To tag/invalidate by entity or model **class/instance**, use the
`Contao\CoreBundle\Cache\CacheTagManager` service (the doc refers to it as the
"EntityCacheTags helper"): `tagWithModelInstance(...)`,
`invalidateTagsForModelClass(...)`, etc.

## Caching fragments

Content elements and front end modules are **fragment controllers**; each
returns a `Response` that declares its own cacheability. Two render modes:

- **Inline** (default) — rendered inside the main request and merged before
  caching. Page and fragment cache times merge to the **lowest common
  denominator** (page cacheable a day, fragment an hour → whole page an hour).
- **Subrequest / ESI** — rendered *after* the page is cached, so the fragment
  can have its own (e.g. longer) cache time; the reverse proxy merges the
  pieces.

### Inline fragments

Set cache info on the template response inside `getResponse()`:

```php
$response = $template->getResponse();
$response->setPublic();
$response->setMaxAge(3600);
return $response;
```

A bare Symfony `Response` is `Cache-Control: private` by default; to let an
inline fragment affect the page cache time it must carry the
`Contao-Merge-Cache-Control` header. This is applied automatically for
responses generated from a Contao template, so prefer `$template->getResponse()`.

### ESI fragments

Set the fragment renderer to `esi` to cache the fragment **separately** from
the page. Only worthwhile when the fragment itself is **cacheable** and
**expensive to generate** (e.g. a once-a-day weather API call). For cheap
fragments, cache the whole page and regenerate more often instead — ESI on a
private/uncacheable fragment forces the proxy to boot the full system on every
request and can **worsen** performance. Symfony auto-detects ESI support and
falls back to inline rendering if the proxy lacks it.

More detail and full examples: [references/fragments.md](references/fragments.md).

[fos]: https://foshttpcachebundle.readthedocs.io/en/latest/
[store]: https://github.com/Toflar/psr6-symfony-http-cache-store
[cb]: https://docs.contao.org/dev/reference/dca/callbacks/#config-oninvalidate-cache-tags
