---
name: sulu-extend-entities
description: Create and extend Doctrine entities in Sulu — the Persistence component, entity extensions, dimension content, and extensible entities. Triggers when adding custom entities, extending Sulu core entities, or implementing dimension/extensible content.
---

# Extending entities in Sulu

Sulu uses Doctrine entities together with its **Persistence** infrastructure so
that entities can be replaced or extended without rewriting the code that uses
them. There are four distinct tasks, each with its own pattern:

1. **Extend a built-in Sulu entity** (User, Role, Contact, Account, Category,
   Media, Tag) via the per-bundle `objects` config.
2. **Extend dimension content** (Sulu 3.0 Page / Article / Snippet) — a
   two-part model with custom mappers/mergers.
3. **Make your own entity extensible** with the Persistence bundle
   (interface + repository + DI wiring).
4. Understand the **Persistence component** primitives all of the above use.

Pick the matching section. Long, complete code samples live in the linked
reference files; keep new code aligned with the rules below.

## 1. Extend a built-in Sulu entity

Sulu can replace these internal entities the same way:
**User, Role, Contact, Account, Category, Media, Tag**.

Rules:

- Create an entity in `App\Entity` that **extends** the Sulu class
  (e.g. `Sulu\Bundle\SecurityBundle\Entity\User`).
- The `#[ORM\Table(name: ...)]` on your entity **must match the original
  table** (e.g. `se_users`, `co_contacts`, `me_media`, `ta_tags`, …).
  A mismatched table name causes Doctrine query errors.
- Own properties **must have sensible default values**, otherwise core
  features (such as `sulu:security:user:create`) can crash.
- Point Sulu at your class through the bundle's `objects.<name>.model`
  (and optionally `.repository`) config key, then `cache:clear`.
- When replacing entities in an **existing** project, migrate existing data to
  avoid data loss.

Per-entity config keys (model / repository) and a full example:
see [references/built-in-entities.md](references/built-in-entities.md).

## 2. Extend dimension content (Page / Article / Snippet, Sulu 3.0)

In Sulu 3.0 content resources are split into two parts:

- the **content-rich entity** (`Page`, `Article`, `Snippet`) — holds values
  that are **global** to the resource and do not vary by locale/stage/version
  (page-tree position, webspace, identifiers);
- one or more **`DimensionContent`** records — locale-, stage- and
  version-aware editable content (editorial metadata, custom tabs, queryable
  content fields). This is where most editor-managed fields belong.

A real extension typically requires:

1. Replace **both** models in config — `<resource>.model` **and**
   `<resource>_content.model` (replacing only the dimension content is not
   enough).
2. The content-rich entity must return your custom class from
   `createDimensionContent()` (Sulu creates missing localized/unlocalized
   dimension contents through it — mandatory).
3. Add DB columns / relation tables on the dimension content entity, keeping
   the original table names (`pa_pages` / `pa_page_dimension_contents`,
   `ar_articles` / `ar_article_dimension_contents`,
   `sn_snippets` / `sn_snippet_dimension_contents`).
4. A **`ContentDataMapper`** (`sulu_content.data_mapper`) — Sulu only persists
   template data automatically; every dedicated property needs a mapper.
   Write to the **localized** dimension content for per-locale values, to the
   **unlocalized** one for values shared across locales.
5. A **`ContentMerger`** (`sulu_content.merger`) for custom fields that must
   survive aggregation, workflow transitions, and copy/restore.

Add a **`ContentNormalizer`** (`sulu_content.normalizer`) **only** when the API
output must differ from default getter serialization (nested form data,
relation IDs, computed fields). Sulu serializes getters with Symfony's
`GetSetMethodNormalizer` first, then applies tagged normalizers; this same
normalization path is used for copy/restore.

Storage choice:

| Put it on… | When |
|---|---|
| content-rich entity | value is global, not locale/stage/version-bound (rare for editor fields) |
| dimension content, scalar column | lifecycle value used in filters, sorting, indexes (status, priority, dates) |
| dimension content, relation | value has own identity/lifecycle or needs referential integrity (m:n, reusable sets) |
| dimension content, JSON column | grouped low-query settings / nested tab data |

Do not hide query-critical fields in JSON — keep dedicated columns for fields
you filter, sort, or index.

Full entity, mapper, merger, normalizer and service-tag examples:
see [references/dimension-content.md](references/dimension-content.md).

## 3. Make your own entity extensible (Persistence bundle)

To let downstream bundles replace your entity, build it on an interface +
repository and wire it through the Persistence bundle. Using a `Book` example,
you need:

- **`BookInterface`** — the type used **everywhere** (variables, Doctrine
  relations, type hints) so the implementation stays exchangeable.
- **`Book`** — default implementation, mapped as a **mapped-superclass**, the
  base class for extension.
- **`BookRepositoryInterface`** — extends the Persistence
  `RepositoryInterface`; that base interface defines **`createNew()`**, which
  you must use instead of `new` so you never instantiate the wrong
  implementation.
- **`BookRepository`** — implements the repository interface and should extend
  the Persistence `EntityRepository`, which provides a dynamic `createNew()`
  returning the currently configured implementation.

Three wiring points:

- **`Configuration`** declares default `model` / `repository` under
  `sulu_book.objects.book.*`, giving override paths.
- **`Extension`** uses `PersistenceExtensionTrait::configurePersistence()` on
  `$config['objects']`, then `addAliases()` to alias the repository interface
  to the configured repository service.
- **`Bundle::build()`** uses `PersistenceBundleTrait::buildPersistence()` to
  map the entity interface to its class parameter (adds a
  `ResolveTargetEntitiesPass`), so the interface resolves in Doctrine mappings.

The Persistence bundle then provides for `book`:

- `sulu.model.book.class` — parameter, current entity class
- `sulu.repository.book` — service, current repository
- `BookRepositoryInterface` — autowire alias to that repository service

Full interface/repository/Configuration/Extension/Bundle code:
see [references/extensible-entity.md](references/extensible-entity.md).

## 4. Persistence component primitives

The shared infrastructure under `Sulu\Component\Persistence` and
`Sulu\Bundle\PersistenceBundle`:

- **`RepositoryInterface`** — declares `createNew()`; never construct managed
  entities with `new` in extensible code, always go through the repository so
  the configured implementation is returned.
- **`EntityRepository`** (ORM) — base repository implementing a dynamic
  `createNew()`.
- **`PersistenceExtensionTrait`** / **`PersistenceBundleTrait`** — the DI/build
  glue that registers the `sulu.model.*.class` parameters and
  `sulu.repository.*` services and resolves interfaces to implementations.

Use these primitives rather than hand-rolling factories or hard-coding entity
classes; that is what keeps `objects.*.model` / `.repository` overrides working
end to end.

## Always

- `cache:clear` (admin context) after changing `objects` config.
- Match original table names when replacing built-in entities.
- Type against interfaces, create via `createNew()`, in extensible code.
- Replace **both** the content-rich and dimension-content models together.
