---
name: tinymce-configuration
description: Configuring a TinyMCE editor through its option system. Use when editing a tinymce.init({...}) config object, choosing toolbar/menubar/menu items, setting editor options (selector, inline, plugins, height, content_css, etc.), or reading resolved option values with editor.options.get().
---

# TinyMCE configuration

TinyMCE is configured by passing a single **options object** to `tinymce.init({...})`.
Every option is validated against a registered type before the editor uses it. This skill
covers the most important options, the toolbar/menu item-name model, and how options are
read back at runtime.

## How options work

- Each option has a registered **processor** (type) and often a **default**. Passing a
  value of the wrong type logs a warning and the option falls back to its default — it does
  not throw. So a misspelled or mistyped option is silently ignored.
- Read the *resolved* value of any option at runtime with `editor.options.get('name')`
  (returns the set value, else the registered default, else `undefined`). Related calls:
  `editor.options.set(name, value)`, `unset(name)`, `isSet(name)`, `isRegistered(name)`,
  and `editor.options.debug()` to log the raw init config.
- Available **options depend on which plugins and model/theme are loaded** — e.g. table,
  image, and paste options only exist once those plugins are enabled. Only document/use
  options that are actually registered.

## Core options

| Option | Type | What it does |
| --- | --- | --- |
| `selector` | string | CSS selector for the element(s) to turn into editors. Use this OR `target`, not both. |
| `target` | object (HTMLElement) | A direct element reference instead of `selector`. |
| `inline` | boolean (default `false`) | `true` edits an element in place (no iframe); `false` uses the classic iframe editor. |
| `plugins` | string[] (default `[]`) | Plugins to load. Accepts an array or a space/comma-separated string. |
| `toolbar` | boolean \| string \| string[] \| ToolbarGroup[] | Toolbar buttons. See item-name model below. `false` hides the toolbar. |
| `toolbar_mode` | `'floating'` \| `'sliding'` \| `'scrolling'` \| `'wrap'` | How overflowing toolbar buttons behave. |
| `toolbar_location` | `'top'` \| `'bottom'` \| `'auto'` | Where the toolbar sits. |
| `menubar` | boolean \| string | `false` hides the menu bar; a string picks which top-level menus show (e.g. `'file edit view'`). |
| `menu` | object | Customize/define menus: `{ name: { title, items } }` where `items` is a space-separated item string. |
| `height` | number \| string | Editor height (px number or any CSS length string). |
| `width` | number \| string | Editor width (px number or any CSS length string). |
| `content_css` | boolean \| string \| string[] (default `['default']`, `[]` for inline) | Stylesheet(s) applied **inside** the editable content. `false` loads none. |
| `content_style` | string | Raw CSS injected into the content area — handy for one-off rules without a file. |
| `placeholder` | string | Placeholder text shown when empty (defaults to the source element's `placeholder` attribute). |
| `readonly` | boolean (default `false`) | Renders content non-editable. (See also `disabled`.) |
| `statusbar` | boolean | Show/hide the bottom status bar. |
| `branding` | boolean | Show/hide the "Powered by Tiny" branding. |
| `resize` | boolean \| `'both'` | Status-bar resize handle: `false` off, `true` vertical, `'both'` vertical + horizontal. |
| `skin` | boolean \| string | UI skin name; `false` loads no skin (you supply CSS). Pair with `skin_url` for a custom path. |
| `language` | string (default `'en'`) | UI language code (needs the matching language pack; see `language_url`). |
| `setup` | function | `(editor) => {...}` callback run early — register buttons, bind events before init completes. |
| `base_url` | string | Base URL TinyMCE resolves its own resources from. |
| `suffix` | string | Filename suffix for TinyMCE resources, e.g. `'.min'` to load minified files. |

`base_url`/`suffix` are normally set automatically; only override them when self-hosting
with a non-standard layout. `branding` and `resize` are UI options provided by the theme.

## Toolbar & menu item-name model

Toolbar and menu contents are **space-separated item names**, with `|` as a visual
separator between groups:

```js
tinymce.init({
  selector: '#editor',
  toolbar: 'undo redo | bold italic underline | bullist numlist | link'
});
```

- Each name (`bold`, `link`, `bullist`, …) is a button/menu item contributed by **core or
  an enabled plugin**. A name that no plugin registers simply doesn't appear.
- **Multiple toolbar rows**: pass an **array of strings** (one string per row) —
  `toolbar: ['undo redo | bold italic', 'link image | table']` — or use the numbered
  `toolbar1`, `toolbar2`, … `toolbar9` options.
- **Named groups**: pass an array of `{ name?, label?, items: string[] }` objects for
  structured grouping.
- **Menus** use the same item-name strings inside `menu`:

```js
tinymce.init({
  selector: '#editor',
  menubar: 'file edit format',
  menu: {
    format: { title: 'Format', items: 'bold italic underline | removeformat' }
  }
});
```

## Rules of thumb

- Set **`selector` or `target`, never both**. `selector` is the common choice.
- Only put a button/menu name in `toolbar`/`menu` if the plugin that provides it is in
  `plugins`. Listing `link` without the `link` plugin yields no button.
- Prefer `content_css` (a stylesheet) for real styling; use `content_style` only for a few
  inline rules. Neither affects the surrounding page — they apply to editor content.
- `readonly: true` makes content non-editable; the related `disabled` option toggles a
  disabled state. Pick the one that matches your intent.
- Keep heavy/one-off logic in `setup(editor)` rather than scattering callbacks.
- To inspect what the editor actually resolved, call `editor.options.get('toolbar')`
  (etc.) or `editor.options.debug()` instead of guessing.

## Minimal example

```js
tinymce.init({
  selector: 'textarea#content',
  height: 500,
  plugins: 'lists link image table',
  toolbar: 'undo redo | bold italic | bullist numlist | link image table',
  menubar: 'file edit insert format table',
  content_style: 'body { font-family: system-ui, sans-serif; font-size: 16px }'
});
```

## Further reading

- [references/options-catalog.md](references/options-catalog.md) — a broader list of
  registered options (content/serialization, paste, images/uploads, URL handling,
  newline/indent, security/sanitization, callbacks) with types and defaults.
- [references/toolbar-recipes.md](references/toolbar-recipes.md) — multi-row toolbars,
  named groups, `toolbar_mode`, inline editing, and `setup` examples.
