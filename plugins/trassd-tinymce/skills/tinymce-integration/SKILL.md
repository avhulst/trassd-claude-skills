---
name: tinymce-integration
description: Initializing and embedding a TinyMCE rich-text editor on a page via tinymce.init(options), managing the editor lifecycle through the global tinymce.* (EditorManager), and wiring it into forms. Use when adding a TinyMCE editor to a page, calling tinymce.init, choosing classic vs inline mode, configuring self-hosted vs CDN loading, or tearing editors down on SPA navigation.
---

# TinyMCE integration

How to load, initialize, and dispose of TinyMCE through its global API. The
global `tinymce` object (also exposed as `tinyMCE`) is the **EditorManager** —
the single entry point for creating and managing every editor instance on the
page.

## Initialize with `tinymce.init(options)`

Create editors by calling `tinymce.init` with one options object that is applied
to every editor it creates. Tell it *which* elements to target with either:

- **`selector`** — a CSS selector; every matching element becomes an editor.
- **`target`** — a single DOM element (`HTMLElement`) to turn into one editor.

Use `selector` for the common case; use `target` only when you already hold the
element reference. Don't pass both — `selector` takes precedence and `target` is
ignored.

```js
// Selector form — turns every matching <textarea> into an editor
tinymce.init({
  selector: 'textarea.rich',
});

// Target form — a single, already-resolved element
tinymce.init({
  target: document.getElementById('editor'),
});
```

### Always await the returned Promise

`tinymce.init` returns a `Promise` that resolves with an **array of the created
`Editor` instances** once they have all finished initializing. Await it (or
`.then` it) before touching editor APIs — the editors do not exist synchronously
when `init` returns.

```js
const editors = await tinymce.init({ selector: 'textarea.rich' });
editors.forEach((editor) => {
  editor.setContent('<p>Ready.</p>');
});
```

If the selector matches nothing (or the targets already have editors), the
Promise resolves with an empty array rather than rejecting — check
`editors.length` before assuming an editor was created.

## Classic (iframe) vs inline editing

By default TinyMCE renders in **classic mode**: the editable content lives in an
isolated `<iframe>`, fully shielded from the host page's CSS. This is the right
default for forms and most embeddings.

Set **`inline: true`** to edit an existing element in place (no iframe), so the
content inherits the surrounding page styles — useful for in-context/WYSIWYG
editing of a live page region.

```js
// Inline editing on a content element
tinymce.init({
  selector: '#article-body',
  inline: true,
});
```

Inline mode cannot attach to replaced/void or table-internal elements. Targets
such as `textarea`, `img`, `input`, `table`, `tr`, `td`, `iframe`, `script`,
`style`, `br`, and similar tags are **invalid inline targets** and are skipped
with an init error. Point inline editors at a normal container element (e.g. a
`div`); keep `textarea` targets for classic mode.

## Loading TinyMCE: self-hosted vs CDN

TinyMCE auto-detects where it was loaded from to resolve its plugins, themes,
and language packs. On load it sets two manager properties:

- **`tinymce.baseURL`** — the root directory TinyMCE was served from.
- **`tinymce.suffix`** — the filename suffix for lazily loaded assets, `.min`
  when the minified `tinymce.min.js` was used, otherwise empty.

Detection works when you load the standard `tinymce(.min).js` script directly.
If you load TinyMCE through a bundler, a renamed file, or a path it can't infer,
set the location explicitly via init options so plugins/themes resolve:

- **`base_url`** — directory where the TinyMCE installation lives.
- **`suffix`** — `'.min'` to load minified assets, or `''` for unminified.

```js
// Self-hosted under a custom path
tinymce.init({
  selector: 'textarea.rich',
  base_url: '/assets/vendor/tinymce',
  suffix: '.min',
});
```

For the Tiny Cloud / CDN build, the script's own URL is the base, so detection
normally suffices; only override `base_url`/`suffix` when you self-host or
relocate the assets.

## App-wide defaults with `overrideDefaults()`

To apply settings to **every** `tinymce.init` call in an application, set them
once with `tinymce.overrideDefaults(options)`. These are merged into each
subsequent `init` call, so you don't repeat boilerplate per editor.

```js
// Run once during app startup, before any init() call
tinymce.overrideDefaults({
  base_url: '/assets/vendor/tinymce',
  suffix: '.min',
  toolbar_sticky: true,
});
```

Notes:
- `overrideDefaults` **replaces** the previous defaults — call it once with the
  full set rather than incrementally; a later call overrides an earlier one.
- A `base_url` passed here updates `tinymce.baseURL`; a `suffix` here updates
  `tinymce.suffix`.
- When extending the cloud-based defaults, spread the existing
  `tinymce.defaultOptions` into your object so required cloud defaults survive:
  `tinymce.overrideDefaults({ ...tinymce.defaultOptions, ...customOptions })`.

## Managing instances through the manager

The global `tinymce` exposes the whole lifecycle:

- **`tinymce.get()`** — array of all live editors.
- **`tinymce.get(idOrIndex)`** — a single editor by its DOM id or numeric
  index, or `null` if absent. (The init id comes from the target element's
  `id`/`name`, or is auto-generated.)
- **`tinymce.activeEditor`** — the currently active editor (or `null`).
- **`tinymce.execCommand(cmd, ui, value)`** — run a command on the active
  editor (also handles manager commands like `mceAddEditor`,
  `mceRemoveEditor`, `mceToggleEditor`); returns whether it executed.
- **`tinymce.majorVersion` / `tinymce.minorVersion`** — the loaded build's
  version, handy for diagnostics and feature gating.

```js
const editor = tinymce.get('article-body');
if (editor) {
  editor.setContent('<p>Hello.</p>');
}
```

## Wiring into a form

In classic mode TinyMCE keeps the underlying `<textarea>` in sync so a normal
form submit works. To force every editor to flush its content back to its source
element before a manual/AJAX submit, call **`tinymce.triggerSave()`**.

```js
form.addEventListener('submit', () => {
  tinymce.triggerSave(); // copy editor HTML into the backing fields
});
```

## Tear down on teardown / SPA navigation

Editors are **not** garbage-collected just because their DOM is removed. In a
single-page app, always remove editors before unmounting the view that hosts
them, or you leak instances and event bindings. Use **`tinymce.remove()`**:

```js
tinymce.remove();            // remove ALL editors
tinymce.remove('textarea');  // remove editors matching a CSS selector
tinymce.remove('#article');  // remove editors matching this selector
tinymce.remove(editor);      // remove a specific Editor instance
```

`remove(selector)` selects DOM elements and removes their bound editors;
`remove(editor)` removes that one instance and returns it (or `null` if it
wasn't tracked). A teardown that matches nothing is a no-op.

```js
// SPA route lifecycle
function mount() {
  return tinymce.init({ selector: '#editor' });
}
function unmount() {
  tinymce.remove('#editor'); // free instances before removing the DOM
}
```

## Rules of thumb

- One `tinymce.init` call configures many editors via `selector`; reuse a single
  options object instead of initializing each element separately.
- Await the `init` Promise before calling editor methods; it resolves with the
  created editors (possibly empty).
- Default to classic mode (iframe) for forms; use `inline: true` only for
  in-page content elements, never for `textarea`/replaced/table targets.
- Let TinyMCE auto-detect `baseURL`/`suffix` when loading the standard script;
  set `base_url`/`suffix` explicitly when self-hosting or bundling.
- Centralize shared settings once with `overrideDefaults()` (full set, called
  once) rather than repeating them per `init`.
- Call `triggerSave()` before manual form submission; call `remove()` on
  teardown to avoid leaking editor instances.
