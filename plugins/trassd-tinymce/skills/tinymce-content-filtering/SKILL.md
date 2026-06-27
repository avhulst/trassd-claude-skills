---
name: tinymce-content-filtering
description: Controls what HTML TinyMCE accepts, cleans, and outputs via its schema and parse/serialize pipeline. Use when restricting or allowing tags, attributes, classes or styles, cleaning pasted or loaded HTML, configuring output formatting, or hardening an editor against unsafe content (XSS, unsafe iframes/embeds, script URLs).
---

# TinyMCE content filtering & sanitization

TinyMCE runs every piece of HTML it loads or pastes through two stages:

1. **`DomParser`** parses the input into an abstract `Node` tree, **sanitizes** it,
   and enforces the **`Schema`** (which elements/attributes are valid, valid
   parent/child relationships, empty-node handling).
2. **`Serializer`** writes that tree back out as the string you get from
   `editor.getContent()` / on form submit.

You configure both through `tinymce.init({ ... })` options. The live instances
are reachable as `editor.schema`, `editor.parser` (the `DomParser`) and
`editor.serializer`.

## Security first — rules of thumb

- **Sanitization is ON by default and must stay on.** `xss_sanitization`
  defaults to `true`. Do **not** set it to `false` unless you fully control the
  HTML and have an overriding reason — turning it off disables the built-in XSS
  cleaning entirely.
- **`sandbox_iframes` defaults to `true`** and **`convert_unsafe_embeds`
  defaults to `true`** — keep them on. Sandboxing forces a restrictive sandbox
  on iframes; embed conversion turns unsafe `<object>`/`<embed>` into safer
  equivalents. Only widen via `sandbox_iframes_exclusions` for hosts you trust
  (the default exclusion list already covers YouTube, Vimeo, Spotify, etc.).
- **`allow_script_urls` defaults to `false`** — leaving it false blocks
  `javascript:` and similar URLs. Setting it `true` is a direct XSS vector;
  avoid it. `allow_unsafe_link_target` (default `false`) keeps
  `target="_blank"` links safe with `rel="noopener"`; leave it false.
- **`allow_html_in_named_anchor` defaults to `false`.** Leave it false unless
  you specifically need HTML inside named anchors.
- **Allowlist, don't denylist.** Prefer `valid_elements` /
  `extended_valid_elements` (explicit allow) over `invalid_elements` (block
  list), which is easy to bypass.
- **Never trust the client.** Whatever you configure here, **always
  re-sanitize on the server when saving.** Client-side filtering is for UX and
  defense-in-depth, not a security boundary — a user can bypass the editor
  entirely.

## Controlling allowed HTML

| Option | Type | What it does |
|---|---|---|
| `valid_elements` | string | **Replaces** the whole allowlist of elements/attributes (rule-string syntax below). |
| `extended_valid_elements` | string | **Extends** the default schema with extra rules (same syntax). Preferred for additive changes. |
| `invalid_elements` | string | Comma/space list of elements to delete from the schema. |
| `valid_children` | string | Adjust which children an element may contain (`+`/`-`/replace). |
| `valid_classes` | string \| object | Restrict allowed `class` values, optionally per-element. |
| `valid_styles` | string \| object | Restrict allowed inline `style` properties, per-element or `*`. |
| `invalid_styles` | string \| object | Block specific inline `style` properties. |
| `custom_elements` | string \| object | Register non-HTML / web-component elements in the schema. |
| `schema` | string | Base schema preset: `'html5'` (default) or `'html4'`. |
| `verify_html` | boolean | Default `true`. **Setting it `false` allows all elements/attributes** (`*[*]`) — do not do this for untrusted content. |

Short examples:

```js
// Lock the editor down to a minimal, safe subset (replaces the default allowlist)
tinymce.init({
  selector: '#editor',
  valid_elements: 'p,strong/b,em/i,a[href|title],ul,ol,li,br'
});

// Keep defaults but additively allow a few extras
tinymce.init({
  selector: '#editor',
  extended_valid_elements: 'span[class|style],img[src|alt|width|height]'
});

// Restrict classes and inline styles
tinymce.init({
  selector: '#editor',
  valid_classes: { '*': 'highlight intro', a: 'btn btn-primary' },
  valid_styles: { '*': 'color font-weight text-align' }
});
```

The full **`valid_elements` rule-string syntax** (`@` shared attrs, `!`
required, `=`/`~`/`<` attribute values, `*?+` wildcards, `#`/`-` empty-handling,
`/` substitution) plus `valid_children`, `valid_classes`/`valid_styles` and
`custom_elements` details are in
[references/valid-elements-syntax.md](references/valid-elements-syntax.md).

## Output formatting

| Option | Default | Effect |
|---|---|---|
| `entity_encoding` | `'named'` | How characters are entity-encoded on output (e.g. `named`, `numeric`, `raw`). |
| `element_format` | `'html'` | `'html'` or `'xhtml'` output style (e.g. self-closing void tags). |
| `convert_urls` | `true` | Master switch for URL rewriting on output. |
| `relative_urls` | `true` | Convert absolute URLs to document-relative ones. |
| `remove_script_host` | `true` | When converting to absolute, drop the host so URLs stay root-relative. |

```js
tinymce.init({
  selector: '#editor',
  entity_encoding: 'raw',
  element_format: 'xhtml',
  convert_urls: false   // store URLs exactly as authored
});
```

## The parse / serialize pipeline & node filters

`DomParser` and `Serializer` both take the `Schema`; `DomParser` additionally
runs sanitization. You normally don't construct them — use the editor's
instances and register **filters** to transform nodes during parsing/serializing:

- `editor.parser.addNodeFilter(name, cb)` / `addAttributeFilter(name, cb)` —
  run while loading/pasting.
- `editor.serializer.addNodeFilter(...)` / `addAttributeFilter(...)` — run on
  output.

`name` is a comma-separated list of node names (for `addNodeFilter`) or
attribute names (for `addAttributeFilter`). The callback receives
`(nodes, name, args)` where each entry is a `tinymce.html.Node`. Use
`node.attr('x', value)` / `node.attr('x', null)` to set/remove attributes,
`node.remove()` / `node.unwrap()` to drop nodes.

```js
tinymce.init({
  selector: '#editor',
  setup: (editor) => {
    // Strip any leftover on* event-handler attributes as defense-in-depth
    editor.parser.addAttributeFilter('onclick,onload,onerror', (nodes, name) => {
      nodes.forEach((node) => node.attr(name, null));
    });
  }
});
```

A deeper walkthrough of the `Node` API, the parser's empty-node / whitespace
handling, and how the schema drives validation lives in
[references/pipeline-and-schema.md](references/pipeline-and-schema.md).

## Quick checklist

- Use `extended_valid_elements` to add, `valid_elements` to fully replace; both
  use the same rule syntax.
- Leave `xss_sanitization`, `sandbox_iframes`, `convert_unsafe_embeds` at their
  `true` defaults; leave `allow_script_urls`, `allow_unsafe_link_target`,
  `allow_html_in_named_anchor` at `false`.
- Never set `verify_html: false` for untrusted content.
- Prefer allowlists over `invalid_elements`.
- Re-sanitize server-side on save, every time.
