# Parse / serialize pipeline, the `Node` tree & the `Schema`

## Overview

TinyMCE represents content internally as a tree of `tinymce.html.Node`
(abstract syntax nodes), distinct from the live browser DOM. Two classes move
content in and out of that tree:

- **`tinymce.html.DomParser`** — `parse(html, args)` turns an HTML string into a
  `Node` tree. It sanitizes the input, applies the `Schema`, fixes invalid
  parent/child nesting, normalizes whitespace, and pads/removes empty nodes.
- **`tinymce.html.Serializer`** — `serialize(node)` writes a `Node` tree back to
  an HTML string (this is what `editor.getContent()` ultimately produces).

Both are constructed with a `Schema`. On an editor you use the existing
instances: `editor.parser`, `editor.serializer`, and `editor.schema`. The
parser is created with sane defaults — `validate: true`, `sanitize: true`,
`root_name: 'body'`.

## What `DomParser.parse` does (in order)

1. Parses + **sanitizes** the HTML in a context root, applying the configured
   security options.
2. Builds the `Node` (AST) tree from the sanitized DOM.
3. Normalizes whitespace (collapses runs, trims at block edges).
4. When `validate` is on, finds **invalid children** per the schema and cleans
   them (unwraps/splits so the tree is schema-valid), e.g. it turns
   `<p>a<p>b</p>c</p>` into three sibling `<p>` elements.
5. Pads or removes empty elements according to each element's schema rule
   (`paddEmpty`, `removeEmpty`, `paddInEmptyBlock`).
6. Wraps stray inline/text content into the forced root block when the root is
   `body`.
7. Runs registered **node/attribute filters** (only when the content is valid).

## The `Node` API (`tinymce.html.Node`)

Each node has `name`, `type` (1 = element, 3 = text, 8 = comment, …),
`attributes`, `value`, and tree links (`parent`, `firstChild`, `lastChild`,
`next`, `prev`). Useful methods:

| Method | Purpose |
|---|---|
| `attr(name)` | Get an attribute value. |
| `attr(name, value)` | Set an attribute; `attr(name, null)` removes it. |
| `attr(obj)` | Set several attributes from a name/value map. |
| `append(node)` / `insert(node, ref, before?)` | Add children. |
| `wrap(wrapper)` / `unwrap()` | Wrap node, or remove node keeping its children. |
| `replace(node)` | Replace this node with another. |
| `remove()` | Detach this node from its parent. |
| `empty()` | Remove all children. |
| `getAll(name)` / `children()` | Collect descendants by name / direct children. |
| `isEmpty(elements, whitespace?, predicate?)` | Whether the node counts as empty. |
| `clone()` | Shallow clone (drops `id`). |
| `walk(prev?)` | Step to the next/previous node in tree order. |

Create nodes with `new tinymce.html.Node(name, type)` or
`tinymce.html.Node.create(name, attrs)`.

## Node & attribute filters

Filters are how you hook custom transformation into the pipeline without
touching the schema. Register them on `editor.parser` (load/paste path) and/or
`editor.serializer` (output path):

- `addNodeFilter(name, callback)` — `name` is a comma-separated list of node
  names; the callback gets `(nodes, name, args)`, where `nodes` is every matched
  `Node` collected during the walk.
- `addAttributeFilter(name, callback)` — same, but matches nodes that **have**
  any of the comma-separated attribute names.
- `removeNodeFilter(name, callback?)` / `removeAttributeFilter(name, callback?)`
  — remove a specific callback, or all filters for those names if omitted.

```js
tinymce.init({
  selector: '#editor',
  setup: (editor) => {
    // On output: force every external link to open safely
    editor.serializer.addNodeFilter('a', (nodes) => {
      nodes.forEach((node) => {
        if (node.attr('target') === '_blank') {
          node.attr('rel', 'noopener noreferrer');
        }
      });
    });

    // On input: collect <img> by src and drop tracking pixels
    editor.parser.addAttributeFilter('src', (nodes) => {
      nodes.forEach((node) => {
        if (node.name === 'img' && node.attr('width') === '1') {
          node.remove();
        }
      });
    });
  }
});
```

Filters run **after** sanitization and schema validation, so they are a
defense-in-depth and shaping layer — not a substitute for keeping
`xss_sanitization` and the schema allowlist tight.

## The `Schema` (`editor.schema`, `tinymce.html.Schema`)

The schema is built from the `schema` preset (`'html5'` default, or `'html4'`)
plus your `valid_elements` / `extended_valid_elements` / `invalid_elements` /
`valid_children` / `custom_elements` options. Setting `verify_html: false`
internally sets `valid_elements` to `*[*]` (everything allowed) — avoid for
untrusted content.

Useful read-only query methods:

| Method | Returns |
|---|---|
| `isValid(name, attr?)` | Whether an element (and optional attribute) is allowed. |
| `isValidChild(parent, child)` | Whether `child` may nest in `parent`. |
| `getElementRule(name)` | The compiled rule object for an element (or `undefined`). |
| `isBlock(name)` / `isInline(name)` | Block vs inline classification. |
| `getBlockElements()` / `getVoidElements()` / `getWhitespaceElements()` … | Lookup maps for element classes. |
| `getValidStyles()` / `getValidClasses()` / `getInvalidStyles()` | Compiled style/class allowlists. |
| `getCustomElements()` | Registered custom elements. |

And mutators (used internally by the options, available if you must adjust at
runtime): `addValidElements(str)`, `setValidElements(str)`,
`addValidChildren(str)`, `addCustomElements(strOrObj)`.

```js
if (editor.schema.isValidChild('p', 'span')) { /* span allowed inside p */ }
if (!editor.schema.isValid('script')) { /* script not in the allowlist */ }
```

Whenever you change the schema, the change applies to subsequently parsed
content; the serializer uses the same schema to decide attribute ordering and
void-element handling on output.
