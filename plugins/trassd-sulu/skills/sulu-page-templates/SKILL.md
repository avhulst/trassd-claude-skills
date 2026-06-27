---
name: sulu-page-templates
description: Defines Sulu page and snippet templates the recommended way — the XML structure file (properties, blocks, sections, meta) paired with its Twig view, using the right property types. Triggers when creating or editing templates under config/templates/pages or config/templates/snippets, or their templates/pages/*.html.twig counterparts.
---

# Sulu Page & Snippet Templates

In Sulu a page template controls both the **structure** of a page (its
*properties*) and **how that structure is rendered**. Every template is two
paired files:

- an **XML file** defining the structure — `config/templates/pages/<key>.xml`
- a **Twig file** rendering it — `templates/pages/<key>.html.twig`

Snippets work identically under `config/templates/snippets/` and
`templates/snippets/`.

## Core rules

- **Keep the XML and Twig file in sync.** The XML `<view>` element names the
  Twig file (without the format/extension suffix); Sulu appends
  `.<format>.twig` automatically based on the requested format (html, xml,
  json, …). So `<view>pages/event</view>` resolves to
  `templates/pages/event.html.twig` for an HTML request.
- **The `<key>` must equal the filename minus `.xml`.** It is the unique
  template identifier.
- **Every property `name` is accessed in Twig via `content.<name>`.** A
  `text_line` named `title` is `{{ content.title }}`. Names must be unique
  within a template.
- **Each property needs `name`, `type`, and a `<title>` in `<meta>`.** The
  meta title is the label shown to content managers; provide one per language
  (`lang="en"`, `lang="de"`, …).
- **A page minimally needs `title` (text_line) and `url` (route).** The `url`
  property carries the `sulu.rlp` tag and `title` typically the `sulu.rlp.part`
  tag so Sulu can build the resource locator.
- **Properties are optional by default.** Add `mandatory="true"` to require a
  value. Do not combine `mandatory` with `visibleCondition`/`disabledCondition`
  — conditions are evaluated client-side (JEXL) and cannot be expressed in the
  server-side JSON Schema validation.

## XML skeleton

Every template uses the same namespace and schema:

```xml
<?xml version="1.0" ?>
<template xmlns="http://schemas.sulu.io/template/template"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://schemas.sulu.io/template/template http://schemas.sulu.io/template/template-1.0.xsd">

    <key>event</key>
    <view>pages/event</view>
    <controller>Sulu\Content\UserInterface\Controller\Website\ContentController::indexAction</controller>

    <meta>
        <title lang="en">Event</title>
    </meta>

    <properties>
        <property name="title" type="text_line" mandatory="true">
            <meta><title lang="en">Title</title></meta>
            <tag name="sulu.rlp.part"/>
        </property>

        <property name="url" type="route" mandatory="true">
            <meta><title lang="en">URL</title></meta>
            <tag name="sulu.rlp"/>
        </property>

        <property name="article" type="text_editor">
            <meta><title lang="en">Article</title></meta>
        </property>
    </properties>
</template>
```

The template-level `<meta><title>` is the name shown in the admin template
dropdown.

> In Sulu 3 the default page controller is
> `Sulu\Content\UserInterface\Controller\Website\ContentController`. The older
> `Sulu\Bundle\WebsiteBundle\Controller\DefaultController` (and
> `WebsiteController`) are removed — use `ContentController` instead.

## Common property types

| Type | Use for | Twig access |
|------|---------|-------------|
| `text_line` | plain single-line string | `{{ content.title }}` |
| `text_area` | multiline plain text | `{{ content.text }}` |
| `text_editor` | rich text stored as HTML | `{{ content.article\|raw }}` |
| `single_media_selection` / `media_selection` | one / many media items | see images example below |
| `route` | the page URL | used by Sulu routing |
| `smart_content` | dynamic, filtered item lists | `{{ content.<name> }}` + `{{ view.<name> }}` |
| `block` | repeatable, content-manager-ordered groups | `{% for block in content.<name> %}` |

Many types are configured via a `<params>` element (e.g. a `single_select`
needs a `values` collection; `single_media_selection` takes `types`,
`displayOptions`, etc.).

**text_editor** is text-only by design — Sulu separates content from
presentation, so do **not** embed images in the editor. Handle images as
separate media properties (or inside blocks), and always render the editor
output with the `raw` filter.

## Blocks

Blocks let content managers add and reorder typed sub-content. Use `<block>`
instead of `<property>`; the `default-type` attribute is mandatory, and the
`<types>` element lists the selectable block types. Each `<type>` holds its own
`<properties>` (any property type is allowed). Note: *block types* are your own
named types — distinct from *property types*.

```xml
<block name="blocks" default-type="text_image" minOccurs="0">
    <meta><title lang="en">Content</title></meta>
    <types>
        <type name="text_image">
            <meta><title lang="en">Editor with image</title></meta>
            <properties>
                <property name="title" type="text_line">
                    <meta><title lang="en">Title</title></meta>
                    <tag name="sulu.block_preview" priority="128"/>
                </property>
                <property name="description" type="text_editor">
                    <meta><title lang="en">Article</title></meta>
                </property>
                <property name="image" type="single_media_selection">
                    <meta><title lang="en">Image</title></meta>
                    <params>
                        <param name="types" value="image"/>
                    </params>
                </property>
            </properties>
        </type>
    </types>
</block>
```

Optional attributes: `minOccurs` / `maxOccurs` to bound the number of blocks;
params `add_button_text`, `collapsable`, `movable`. The `sulu.block_preview`
tag (with optional `priority`) picks which fields show in a collapsed block.

Render blocks by looping and dispatching on `block.type`, passing both the
`content` and the matching `view` entry:

```twig
{% for block in content.blocks %}
    {% include 'includes/blocks/' ~ block.type ~ '.html.twig' with {
        content: block,
        view: view.blocks[loop.index0],
    } %}
{% endfor %}
```

Block settings (e.g. a custom theme field added via a
`content_block_settings` form) are available under `block.settings`.

For a block type reused across templates, define it as a **global block**: a
file in `config/templates/blocks/` with its own `<key>`, referenced with
`<type ref="..."/>` inside a `<block>`.

## Sections

Sections group properties **visually in the admin only** — they have no effect
on the data model, so property names are accessed in Twig exactly as if the
section did not exist. A section has a `name`, a `<meta><title>`, and a nested
`<properties>` element.

```xml
<section name="organizationalDetails">
    <meta><title lang="en">Organizational Details</title></meta>
    <properties>
        <property name="startDate" type="date" colspan="6">
            <meta><title lang="en">Start Date</title></meta>
        </property>
        <property name="endDate" type="date" colspan="6">
            <meta><title lang="en">End Date</title></meta>
        </property>
    </properties>
</section>
```

## Useful property attributes & extras

- **`colspan`** — width on the admin's 12-column grid; narrower fields float
  next to each other.
- **`multilingual="false"`** — store one value across all languages (e.g.
  article numbers). Changing this later behaves like a rename and needs data
  migration.
- **`<info_text lang="...">`** inside `<meta>` — help text shown next to a
  field.
- **`visibleCondition` / `disabledCondition`** — JEXL expressions over other
  field values; use `__parent.` to reference a parent scope inside a block, and
  `AND` / `OR` (not `&&` / `||`, which need XML escaping).
- **`<cacheLifetime type="seconds">N</cacheLifetime>`** — sets the client
  `Cache-Control: max-age`; `type="expression"` accepts a cron expression.
- **Search:** tag a property with `<tag name="sulu.search.field" role="..."/>`
  (roles `title`, `description`, `image`) to index it. Roles are not allowed on
  properties inside a block.
- **Reuse:** include shared fragments with XInclude
  (`xmlns:xi="http://www.w3.org/2001/XInclude"`, `<xi:include href="..."/>`),
  optionally targeting parts with an `xpointer`.

## Smart content

A `smart_content` property renders a filtered, content-manager-configured list.
Configure the source provider and which item properties are exposed via
`<params>`:

```xml
<property name="pages" type="smart_content">
    <meta><title lang="en">Latest pages</title></meta>
    <params>
        <param name="provider" value="pages"/>
        <param name="max_per_page" value="5"/>
        <param name="properties" type="collection">
            <param name="article" value="article"/>
            <param name="excerptTitle" value="excerpt.title"/>
        </param>
    </params>
</property>
```

Built-in providers include `pages`, `snippets`, `articles`, `media`,
`contacts`, `accounts`. The resolved items are in `content.<name>`; filter
metadata (pagination, `presentAs`, `total`, …) is in `view.<name>`. For pages
and articles, `title` and `url` are always available without explicit mapping.

```twig
{% for page in content.pages %}
    <a href="{{ sulu_content_path(page.url) }}">{{ page.title }}</a>
{% endfor %}
```

## Rendering in Twig

Sulu passes these default variables to the view:

- **`content`** — values of your defined properties (keyed by property name).
- **`view`** — view data for properties (e.g. smart content pagination,
  media `displayOption`).
- **`extension`** — Sulu extension data, notably `extension.seo` and
  `extension.excerpt` (available on every page regardless of template).
- Request/page metadata: `request.locale`, `request.webspaceKey`, `uuid`,
  `template`, `localizations`, etc.

Examples:

```twig
<h1>{{ content.title }}</h1>
{{ content.article|raw }}

{% for image in content.images %}
    <img src="{{ image.thumbnails['200x100'] }}" alt="{{ image.title }}"/>
{% endfor %}
```

Image thumbnail formats must be defined in your config's image formats file.
Use `{{ dump() }}` (dev mode) to inspect all available variables.

## Snippets

Snippets are reusable content blocks edited independently of pages. Define them
the same way as pages — XML under `config/templates/snippets/`, Twig under
`templates/snippets/` — using the same `<template>` schema, property types,
blocks, and sections. Snippets are selected into pages via the
`snippet_selection` / `single_snippet_selection` property types, and the
`snippets` smart-content provider can filter them (optionally by `type`).
