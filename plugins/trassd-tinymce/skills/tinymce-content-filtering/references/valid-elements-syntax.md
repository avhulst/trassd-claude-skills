# `valid_elements` rule-string syntax

`valid_elements` and `extended_valid_elements` are comma-separated lists of
**element rules**. `valid_elements` (or `Schema.setValidElements`) **replaces**
the allowlist; `extended_valid_elements` (or `Schema.addValidElements`) **adds**
to it. Same syntax for both.

```
[prefix]elementName[/outputName][!][attrPrefix...[attrName][attrValueRule]|...]
```

## Element rules

A single rule names an element and, in `[ ... ]`, the attributes it may carry.
Multiple attributes are separated by `|`.

```
p[class|id]              p may have class and id
a[href|title]            a may have href and title
strong/b                 allow strong; b in the input is OUTPUT as strong (substitution)
img[src|alt|width]       img with these attributes
```

### Element-name prefixes (left of the name)

| Prefix | Meaning |
|---|---|
| `#` | **Pad if empty** — keep the element even when empty, padding it (e.g. with a `<br>`/nbsp). |
| `-` | **Remove if empty** — drop the element when it has no meaningful content. |
| `+` | Used in `valid_children` (see below), not typically as a standalone element prefix here. |

```
#p           keep empty <p> (padded)
-span        remove empty <span>
```

### `!` after the attribute bracket

A `!` placed **immediately before** the attribute bracket marks the element as
"remove empty attrs" — the element is dropped if it has no attributes left:

```
span![class|style]   keep span only while it still has class or style
```

### `*` wildcard / patterns

`*`, `?`, `+` in an element or attribute name make it a **pattern** rather than
an exact match:

- `*` — any element / any attribute name.
- `?` — single character.
- `+` — one or more characters.

```
*[*]            allow ALL elements and ALL attributes (this is exactly what
                verify_html:false sets internally — avoid for untrusted input)
p[data-*]       p may carry any data-* attribute
```

## Attribute value rules

Each attribute inside the brackets may carry a value rule via a one-character
prefix:

| Syntax | Name | Meaning |
|---|---|---|
| `!attr` | **Required** | The attribute MUST be present or the element is invalid. |
| `-attr` | **Denied** | Remove this attribute (e.g. one inherited from the `@` global rule). |
| `attr=value` | **Default value** | If the attribute is missing, add it with `value`. |
| `attr~value` | **Forced value** | Always set the attribute to `value`, overriding the input. |
| `attr<v1?v2?v3` | **Valid values** | The attribute is only kept if its value is one of the listed values (separated by `?`). |

```
a[!href|target=_blank]       href required; target defaults to _blank if absent
img[src|alt~]                src allowed; alt always forced to "" (empty)
td[align<left?center?right]  align only kept when it is left, center or right
@[id|class|-style]           shared global attrs (see below); deny style from it
```

## `@` shared (global) attribute set

A rule named `@` defines a set of attributes shared by **all** subsequent
element rules — its attributes are cloned into every element rule that follows.
Only the first `@` rule is used.

```
@[id|class|title],p[align],a[href]
// p effectively allows id, class, title, align
// a effectively allows id, class, title, href
```

Per-element, you can **deny** a shared attribute with the `-` attribute prefix:

```
@[id|class|style],pre[-style]   // pre allowed id+class but NOT style
```

## `valid_children`

`valid_children` adjusts the allowed parent→child relationships. Comma-separated
rules of the form `name[child1|child2|...]`:

| Prefix on the parent name | Operation |
|---|---|
| (none) | **Replace** the element's allowed-children set. |
| `+` | **Add** the listed children to the existing set. |
| `-` | **Remove** the listed children from the set. |

Children prefixed with `@` are **presets** (named groups of elements) rather
than literal element names.

```js
tinymce.init({
  selector: '#editor',
  // allow <a> and <div> directly inside <p>, keep everything else <p> allows
  valid_children: '+p[a|div]',
  // forbid <div> inside <span>
  // valid_children: '-span[div]'
});
```

## `valid_classes`

Restrict which `class` token values survive. String or per-element object:

```js
tinymce.init({
  selector: '#editor',
  valid_classes: {
    '*': 'highlight lead',     // these classes allowed on any element
    a: 'btn btn-primary'       // a may additionally use these
  }
});
```

## `valid_styles` / `invalid_styles`

Allowlist or denylist inline `style` properties. String (comma/space separated)
or per-element object; `'*'` applies to all elements.

```js
tinymce.init({
  selector: '#editor',
  valid_styles: { '*': 'color background-color font-weight text-align' },
  invalid_styles: 'position'   // never allow position:* inline
});
```

## `custom_elements`

Register non-standard / web-component elements so the schema treats them as
valid. Inline custom elements are named with a leading `~`; others are treated
as block-level. Accepts a comma-separated string or an object with a richer
spec (e.g. `extends`, `attributes`, `children`).

```js
tinymce.init({
  selector: '#editor',
  // ~my-inline is inline; my-widget is block-level
  custom_elements: '~my-inline,my-widget',
  extended_valid_elements: 'my-widget[id|data-config],my-inline[class]'
});
```

> Registering a custom element in `custom_elements` makes it *known* to the
> schema; you still typically declare its allowed attributes via
> `extended_valid_elements` (or the object form's `attributes`).
