---
name: sulu-content-architecture
description: >-
  Understand and work with Sulu's content architecture — structures (templates),
  properties, content types, and smart content / data providers — including how
  content is stored and resolved for the frontend. Use when implementing custom
  content types or selection fields, building a smart-content or page data
  provider, writing PropertyResolvers / ResourceLoaders, or reasoning about how
  Sulu loads and resolves content.
---

# Sulu Content Architecture

How Sulu structures, stores, and resolves content — and the extension points
(smart-content providers, PropertyResolvers, ResourceLoaders) you implement to
add custom data to pages.

## Structures & properties

- Sulu organizes content in **webspaces**, each holding an arbitrary number of
  **pages** ordered in a hierarchical **tree**. The tree *is* the site
  structure — no separate navigation tree is needed; pages can be flagged into
  the navigation.
- Each page can carry content in many **localizations**.
- Every page has a **template** applied. The template defines which
  **properties** the page has; each property is typed by a **property type**,
  which governs the allowed values and configuration of that property.
- Templates are defined in XML (`<template>` with `<properties>`). A property
  declares a `name` and a `type`, plus `<meta>` titles and optional `<params>`.

```xml
<property name="title" type="text_line" mandatory="true">
    <meta><title lang="en">Title</title></meta>
</property>
```

- Advanced page features: **internal links** (redirect to another managed
  page), **external links** (arbitrary URL), and **shadow pages** (reuse the
  content of another localization without re-entering it).

## How content is resolved for the frontend

Content is loaded for rendering by a pipeline centered on the
`ContentResolver`. Understanding it tells you *where* your code plugs in:

1. `ContentResolver` is asked to resolve a page/content entity.
2. For each template property, the matching **PropertyResolver** (selected by
   property type) turns raw data (typically IDs) into a `ContentView`
   containing `ResolvableResource` placeholders.
3. All `ResolvableResource`s are collected into a **priority queue**; entries
   with the same priority and loader key are grouped.
4. The queue is processed by priority, calling the right **ResourceLoader** per
   group — each loader **batch-loads** many resources in a single query
   (avoids N+1).
5. Placeholders are replaced with the loaded data.
6. If loaded resources are themselves content-rich (pages, custom entities),
   they are resolved recursively at the next depth.
7. The fully resolved content is returned for Twig rendering.

PropertyResolvers define **what** to load and **how** to transform;
ResourceLoaders do the **batch loading**. Keep loading out of PropertyResolvers.

## Smart content

Smart content (`smart_content` property type) lets editors dynamically configure
an **aggregation of content** (not only pages). The loading is delegated to
**data providers** registered in the system; Sulu ships providers for pages,
contacts, and accounts.

- Configured in the template via a `smart_content` property. Common params:
  `properties` (a collection mapping Twig keys to page property names;
  `excerpt.` prefix exposes the always-available excerpt extension),
  `present_as` (named display options for a layout dropdown), `max_per_page`
  (pagination), `page_parameter` (GET param when multiple smart contents share
  a page), and `provider` (which data provider to use).
- In Twig, results arrive in `content.<name>` (iterable items) and the
  configuration in `view.<name>` (e.g. `presentAs`, `page`, `hasNextPage`,
  `total`, `maxPage`).

```jinja
{% for page in content.pages %}
  <div class="{{ view.pages.presentAs }}">
    <a href="{{ sulu_content_path(page.url) }}">{{ page.title }}</a>
  </div>
{% endfor %}
```

The configuration array a provider receives can include `dataSource`, `tags` +
`tagOperator`, `categories` + `categoryOperator`, and `types`. Tags/categories
can also be injected from the website via GET params (`websiteTags`,
`websiteCategories`, with their own operators). Providers may additionally
expose `presentAs`, `page`/`pageSize`, and `limit`.

## Implementing a custom SmartContentProvider

Create a service implementing
`Sulu\Bundle\AdminBundle\SmartContent\SmartContentProviderInterface`. It
resolves the configured filters, returns matching items, and advertises a
configuration object. Required methods:

- `getConfiguration(): ProviderConfigurationInterface` — built with
  `Sulu\Bundle\AdminBundle\SmartContent\Configuration\Builder` (e.g.
  `enableTags()`, `enableCategories()`, `enableSorting([...])`,
  `enableTypes([...])`, `enablePagination()`, `enableLimit()`,
  `enableDatasource(...)`, `enableView(...)`).
- `countBy(array $filters, array $params = []): int`
- `findFlatBy(array $filters, array $sortBys, array $params = []): array` —
  **must** return a list of `['id' => ..., 'title' => ...]`. Sulu uses only the
  IDs to decide what to render, then delegates actual loading to the
  ResourceLoader named by `getResourceLoaderKey()`.
- `getType(): string`
- `getResourceLoaderKey(): string`

Register with the `sulu_content.smart_content_provider` tag; the `type`
attribute must match `getType()`, and is what you set as the `provider` param in
the template.

```yaml
services:
    App\SmartContent\ExampleSmartContentProvider:
        tags:
            - { name: 'sulu_content.smart_content_provider', type: 'examples' }
```

For a provider that returns **only pages filtered by a property value**, you can
register a new instance of Sulu's bundled `PageDataProvider` backed by a custom
`QueryBuilder` rather than writing a provider from scratch. (Note: the official
cookbook for this is not yet written for Sulu 3.0.)

Full provider + repository example: [references/smart-content-provider.md](references/smart-content-provider.md).

## PropertyResolvers & ResourceLoaders (custom selection fields)

Use these when building a selection field for a custom entity or integrating an
external data source into a page template.

**PropertyResolver** — implement
`Sulu\Content\Application\PropertyResolver\Resolver\PropertyResolverInterface`:

- `resolve(mixed $data, string $locale, array $params = []): ContentView`
- a `getType(): string` method (used by the tag to index the resolver by
  property type).

Validate input and return an **empty `ContentView`** for invalid data — never
throw. Pick the right `ContentView` factory:
`createResolvablesWithReferences()` (entity selections; enables cache
invalidation via the resource key), `createResolvable()` (single resource), or
`create()` (simple data, no loading). Priority convention: `-50` links/media,
`0` default, `100` articles/snippets, `150` pages (same priority + loader key =
batched). Do **not** load resources here.

**ResourceLoader** — implement
`Sulu\Content\Application\ResourceLoader\Loader\ResourceLoaderInterface`:

- `load(array $ids, ?string $locale, array $params = []): array` — batch-load
  and return results **indexed by ID**; omit missing/inaccessible items
  (handled gracefully). Respect `$locale`; put permission checks here.
- a `getKey(): string` method (used by the tag to index the loader; the key is
  what a PropertyResolver references).

**Service tags & autowiring:** register PropertyResolvers with
`sulu_content.property_resolver` and ResourceLoaders with
`sulu_content.resource_loader`. With autowiring (the Sulu default) these tags
are applied automatically to all implementing services — no manual tagging
needed. The tags index by `getType()` / `getKey()` respectively.

```yaml
services:
    App\Content\PropertyResolver\ProductSelectionPropertyResolver:
        tags: [{ name: 'sulu_content.property_resolver' }]
    App\Content\ResourceLoader\ProductResourceLoader:
        tags: [{ name: 'sulu_content.resource_loader' }]
```

Note: a SmartContentProvider's `getResourceLoaderKey()` points at a
ResourceLoader registered under the same `sulu_content.resource_loader` tag.

Full PropertyResolver + ResourceLoader walkthrough (with Twig output):
[references/property-resolver-and-resource-loader.md](references/property-resolver-and-resource-loader.md).

## Common pitfalls

- Loading resources inside a PropertyResolver — that is the ResourceLoader's
  exclusive job.
- ResourceLoader results not indexed by ID — Sulu cannot match them to the
  resolvables.
- Throwing on bad input in a PropertyResolver instead of returning an empty
  `ContentView`.
- These resolvers/loaders are **read-only, frontend** services. For admin
  interfaces and write operations use Sulu's Admin API and form metadata.
