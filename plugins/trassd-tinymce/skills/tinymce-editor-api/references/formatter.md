# TinyMCE Formatter

Accessed as `editor.formatter`. The formatting engine that backs Bold, italic,
font size, custom styles, etc. It replaces the browser's inconsistent
`execCommand` formatting with a predictable, registry-driven model. Apply formats
through it rather than editing DOM by hand so events and matching stay correct.

## Registering formats

| Method | Notes |
| --- | --- |
| `register(name, format)` | Define a format by name. `format` is a format object (or array of variants). `name` may also be an object mapping several names at once, in which case `format` can be omitted. |
| `unregister(name)` | Remove a registered format. |
| `get(name?)` | Return the named format, or all formats if `name` is omitted. |
| `has(name)` | `true`/`false` if a format is registered under that name. |

```js
editor.formatter.register('highlight', {
  inline: 'span',
  styles: { backgroundColor: '#ffff00' }
});
```

A format can specify `inline` / `block` / `selector` targets, plus `styles`,
`classes`, `attributes`, and may use `%var` placeholders filled from the `vars`
argument at apply time.

## Applying / removing / toggling

All default to the current selection when `node` is omitted.

| Method | Signature | Notes |
| --- | --- | --- |
| `apply` | `apply(name, vars?, node?)` | Apply the format. |
| `remove` | `remove(name, vars?, node?, similar?)` | Remove the format; `node` may be a Node or Range. |
| `toggle` | `toggle(name, vars?, node?)` | Apply if absent, remove if present. |

```js
editor.formatter.apply('highlight');
editor.formatter.toggle('highlight');
editor.formatter.remove('highlight');

// With variables:
editor.formatter.register('fontcolor', { inline: 'span', styles: { color: '%value' } });
editor.formatter.apply('fontcolor', { value: '#ff0000' });
```

## Matching / inspecting

| Method | Returns | Notes |
| --- | --- | --- |
| `match(name, vars?, node?, similar?)` | `boolean` | Does the selection/node have the format. |
| `matchNode(node, name, vars?, similar?)` | format or `undefined` | The matching format object on a specific node. |
| `matchAll(names, vars?)` | `string[]` | Which of the given names match the selection. |
| `closest(names)` | `string \| null` | Closest matching format name from a list. |
| `canApply(name)` | `boolean` | Whether the format can be applied (meaningful mainly for selector formats). |

```js
if (editor.formatter.match('highlight')) { /* selection is highlighted */ }
```

## Reacting to format changes

```js
const { unbind } = editor.formatter.formatChanged('bold,italic', (state, args) => {
  // state: boolean — whether the format is now active
}, /* similar? */ false, /* vars? */ undefined);
// call unbind() to stop listening
```

`formatChanged(formats, callback, similar?, vars?)` watches a comma-separated list
of format names and invokes the callback whenever the selection's match state
changes — ideal for keeping a toolbar button's active state in sync.

## Preview

`getCssText(format)` returns preview CSS text for a format name or inline format
object, e.g. `editor.formatter.getCssText('bold')` or
`editor.formatter.getCssText({ inline: 'b' })`.
