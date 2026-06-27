---
name: sulu-website-rendering
description: >-
  Render the Sulu website layer correctly — custom controllers, Sulu Twig
  attributes/extensions, the request analyzer, and HTTP caching. Use when
  writing website controllers, wiring a page template to a controller via its
  XML <controller> node, passing extra data to page templates, producing custom
  Twig output, rendering a custom route against the page `base` template, or
  configuring shared cache / cache headers / tag-based invalidation for the
  front end.
---

# Sulu website rendering

Guidance for rendering the Sulu front-end (website context): wiring page
templates to controllers, passing extra data, using Sulu's Twig attributes and
extensions, and configuring HTTP caching. Ground every change in the Sulu
WebsiteBundle and HttpCacheBundle behaviour described here.

## Wiring a template to a controller

Each page template declares which controller renders it via the `<controller>`
node in the template XML. The value is `Class::method`:

```xml
<template xmlns="http://schemas.sulu.io/template/template">
    ...
    <controller>App\Controller\Website\CustomController::indexAction</controller>
    ...
</template>
```

Sulu ships a default page controller that resolves the template's property
data and passes it to the Twig template. Configure a custom controller only when
you need to pass **additional** data to the template.

> Stale-symbol caution: Sulu docs name this default as
> `Sulu\Bundle\WebsiteBundle\Controller\DefaultController` and show overriding
> its `getAttributes($attributes, StructureInterface $structure, $preview)`
> method (see `references/custom-controllers.md`). That class name was not
> resolvable in the Sulu 3.x source checkout used to ground this skill — verify
> the exact base-class FQCN and `getAttributes` signature against your installed
> Sulu version before extending it. The resolver-based approach below is the
> verified-portable way to populate page attributes.

## Custom controller — passing extra data

A custom controller may use the full Symfony framework. To pull in services,
make it a service subscriber (implement `getSubscribedServices`) rather than
relying on the container directly. The standard pattern is to extend Sulu's
default page controller and override the attribute-building method so the parent
still resolves all the normal template/property data; you only add your keys.
Full example: `references/custom-controllers.md`.

Rules:
- Always merge into the parent result — never replace the resolved attributes,
  or the template loses its property data.
- Register extra services via `getSubscribedServices()` and call
  `parent::getSubscribedServices()`.

## Rendering a custom route with the page `base` template

For a controller on a **custom route** (not bound to a page) that should reuse
the same `base.html.twig` as pages, build the template attributes with the
`TemplateAttributeResolverInterface` service (`resolve()` takes your custom
parameters and merges them with the Sulu base attributes). Inject the resolver
and Twig, render, and return a `Response`:

```php
use Sulu\Bundle\WebsiteBundle\Resolver\TemplateAttributeResolverInterface;

public function indexAction(
    TemplateAttributeResolverInterface $resolver,
    \Twig\Environment $twig,
): Response {
    return new Response($twig->render(
        'static/custom.html.twig',
        $resolver->resolve(['customAttribute' => 'parameter']),
    ));
}
```

The template can then `{% extends "base.html.twig" %}` and use your keys. Full
example with cache headers: `references/custom-route-base-template.md`.

## Request analyzer & Sulu Twig attributes/extensions

The **request analyzer** (`RequestAnalyzerInterface`) resolves the current
webspace context from the request — it exposes the matched webspace, portal,
segment, current localization, portal URL, resource locator, and arbitrary
per-request attributes (`getWebspace()`, `getPortal()`, `getSegment()`,
`getCurrentLocalization()`, `getPortalUrl()`, `getResourceLocator()`,
`getAttribute()`). This is what populates the `sulu_*` data exposed to
templates.

Sulu adds its own Twig functions and filters on top of the standard Twig set.
Use these in website templates instead of re-implementing content lookups. Full
catalogue: `references/twig-extensions.md`. Highlights:

- **Content/pages (CoreBundle):** `sulu_content_path`, `sulu_content_root_path`,
  `sulu_page_load`, `sulu_page_breadcrumb`, and the navigation helpers
  (`sulu_page_navigation_flat`, `sulu_page_navigation_tree`,
  `sulu_page_navigation_root_flat`, `sulu_page_navigation_root_tree`),
  `sulu_article_load`; filter `sulu_util_multisort`.
- **Snippets:** `sulu_snippet_load_by_area`.
- **Media:** `sulu_resolve_media`, `sulu_resolve_medias`, `sulu_get_media_url`.
- **Tags / categories:** `sulu_tags`, `sulu_tag_url*`, `sulu_categories`,
  `sulu_category_url*`.
- **Contacts / users:** `sulu_resolve_contact`, `sulu_resolve_user`.

## HTTP caching

Caching is provided by the SuluHttpCache bundle, integrating Sulu with HTTP
caching proxies via FOSHttpCacheBundle. Default proxy client is the Symfony
HTTP cache (wraps the kernel); Varnish is also supported.

### Per-response cache headers

Set standard Symfony cache control on the `Response`, and set the
**reverse-proxy TTL** (how long the proxy caches the page) via the Sulu header
constant `SuluHttpCache::HEADER_REVERSE_PROXY_TTL` (verified value
`X-Reverse-Proxy-TTL`):

```php
use Sulu\Bundle\HttpCacheBundle\Cache\SuluHttpCache;

$response->setPublic();
$response->setMaxAge(240);
$response->setSharedMaxAge(240);
$response->headers->set(SuluHttpCache::HEADER_REVERSE_PROXY_TTL, '604800');
```

For per-user / uncached responses, make the response private and disable caching
(`setPrivate()`, `setMaxAge(0)`, `no-cache` / `must-revalidate` / `no-store`
directives) — see `references/custom-route-base-template.md`.

### Cache lifetime from a structure

To apply the cache lifetime configured on a Sulu structure (a `PageInterface`),
use the `CacheLifetimeEnhancer` service (`sulu_http_cache.cache_lifetime.enhancer`)
to `enhance($response, $structure)`. It is only available when a proxy client is
correctly configured — guard with a `has()` check before using it. The
`CacheLifetimeResolver` turns lifetime metadata into an absolute number of
seconds.

### Configuration (defaults)

```yaml
sulu_http_cache:
    cache:
        max_age: 240
        shared_max_age: 240
    tags:
        enabled: false
    proxy_client:
        symfony:
            enabled: false
        varnish:
            enabled: false
    debug:
        enabled: true   # adds an X-Cache: HIT/MISS debug header
```

### Tag-based invalidation

Enable `tags` (requires a proxy client that supports **Banning**). Sulu sends
all UUIDs of structures rendered in a page response as an `X-Cache-Tags` header;
the proxy stores them with the cached HTML. When you edit a structure in the
admin, Sulu instructs the proxy to purge every cache referencing that UUID —
invalidating both the structure's own URLs and pages that reference it. The
rendered entities/documents are collected via the reference store.

For manual invalidation use the `CacheManager`
(`sulu_http_cache.cache_manager`), e.g. `invalidatePath($path, $headers)`.

## Checklist

- Template `<controller>` points at `Class::method`; custom controllers merge
  into (never replace) the resolved attributes.
- Custom routes that reuse `base.html.twig` build attributes with
  `TemplateAttributeResolverInterface::resolve()`.
- Use `sulu_*` Twig functions for content/media/nav rather than custom queries.
- Set both `setSharedMaxAge()` and `HEADER_REVERSE_PROXY_TTL` for proxy caching;
  use private/no-store for user-specific responses.
- Enable `tags` (with a Banning-capable proxy) for reference-aware
  invalidation; use `CacheManager` for explicit purges.
