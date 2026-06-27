---
name: tinymce-editor-api
description: How to drive a live TinyMCE editor instance via its public API — getting the instance, reading/writing content, dirty state and save, commands, events, shortcuts, the UndoManager, and the Formatter. Use when scripting against an editor instance, handling editor events, or programmatically reading or modifying editor content (e.g. wiring a form submit or reacting to changes).
---

# TinyMCE Editor API

Guidance for working with a live `tinymce.Editor` instance after `tinymce.init(...)`.
Everything here is the **runtime** API on the editor object — distinct from the
init-time configuration options.

## Getting an editor instance

Never grab the DOM textarea directly; always go through an editor object.

- **Inside `setup`** — the callback receives the editor before it renders. This
  is the place to register commands, shortcuts, formats and event handlers:
  ```js
  tinymce.init({
    selector: '#content',
    setup: (editor) => {
      editor.on('init', () => { /* editor is ready */ });
    }
  });
  ```
- **By id** — `tinymce.get('content')` returns the instance whose id matches the
  replaced element (or `null` if none). The id equals the original element id.
- **The focused one** — `tinymce.activeEditor` is the most recently focused
  instance. Convenient for one-offs; prefer an explicit `get(id)` in app code
  where several editors may exist.

The editor is only fully usable once initialized. Gate setup-dependent work on
the `init` event (or check `editor.initialized`) rather than running it inline in
`setup`, which fires before render.

## Reading and writing content

These are the methods you reach for most, especially around form submit.

- `editor.getContent()` — returns cleaned **HTML** by default. Pass a format:
  - `editor.getContent({ format: 'html' })` — serialized, cleaned HTML (default).
  - `editor.getContent({ format: 'text' })` — plain text only.
  - `editor.getContent({ format: 'tree' })` — an AST node (advanced).
  - `format: 'raw'` exists for internal/undo serialization; avoid it for
    user-facing reads — it is not cleaned and not meant for storage.
- `editor.setContent(html)` — replaces all content; cleans it first. Accepts the
  same `{ format }` option, e.g. `editor.setContent('<p>Hi</p>', { format: 'html' })`.
- `editor.insertContent(html)` — inserts at the caret (delegates to the
  `mceInsertContent` command). Use this, not `setContent`, to add to existing
  content.
- `editor.resetContent()` — resets content back to the initial start content (or
  to a passed string), and **also** clears the undo history and dirty state. Use
  it after a successful save to establish a fresh baseline.

Idiomatic form integration: read on submit, don't shadow the textarea manually.

```js
form.addEventListener('submit', () => {
  hiddenField.value = tinymce.get('content').getContent();
});
```

`editor.save()` does the standard version of this for you: it serializes the
editor's HTML back into the underlying textarea/element (firing `SaveContent`)
and, unless `set_dirty: false` is passed, resets the dirty flag. Call it before
a native form post if you rely on the original element's value.

## Dirty state

- `editor.isDirty()` — `true` if the user changed content since init or the last
  `save()`. Undo/redo also mark the editor dirty.
- `editor.setDirty(state)` — force the flag. `setDirty(false)` after an
  out-of-band (AJAX) save; setting it `true` fires the `dirty` event once on a
  false→true transition.

Use this to drive "unsaved changes" warnings:

```js
window.addEventListener('beforeunload', (e) => {
  if (tinymce.get('content')?.isDirty()) {
    e.preventDefault();
  }
});
```

## Commands

Commands are the canonical way to mutate content or trigger behaviors.

- `editor.execCommand(cmd, ui?, value?)` — run a command; returns `true` if it
  was handled. `ui` controls whether a dialog is shown.
  ```js
  editor.execCommand('Bold');
  editor.execCommand('mceInsertContent', false, '<strong>Hi</strong>');
  ```
  Note: `execCommand` focuses the editor first (except for a few internal
  commands). Pass `{ skip_focus: true }` as the 4th arg to suppress that.
- `editor.queryCommandState(cmd)` — `true`/`false` toggle state (e.g. is the
  selection bold).
- `editor.queryCommandValue(cmd)` — current value as a string (e.g. font size),
  or `''` if unknown.
- `editor.queryCommandSupported(cmd)` — whether a command is registered.
- `editor.addCommand(name, callback, scope?)` — register a custom command; the
  callback receives `(ui, value, args)`. Pair with
  `addQueryStateHandler` / `addQueryValueHandler` to back a custom toolbar
  toggle.

See [references/commands.md](references/commands.md) for the full command/query API.

## Events

Subscribe with `editor.on(name, handler)`, one-shot with `editor.once(...)`,
detach with `editor.off(...)`. Dispatch your own (or built-in) events with
`editor.dispatch(name, data)`. Handlers receive an `EditorEvent`; calling
`e.preventDefault()` on a `Before*` event cancels the action.

The events you most often want:

- `init` — editor ready; do setup-dependent work here.
- `change` — committed change (a new undo level was added). Use for autosave /
  enabling a save button — it does **not** fire on every keystroke.
- `input` — fires on each input (keystroke-level); use for live counters.
- `SetContent` / `GetContent` — content was set / is being read.
- `NodeChange` — selection moved to a different node; drives toolbar state.
- `dirty` — fired once when the editor first becomes dirty.

```js
editor.on('change', () => saveButton.disabled = false);
editor.on('NodeChange', (e) => updateToolbar(e.element));
```

See [references/events.md](references/events.md) for the full typed event catalog
(content, command, selection, undo/redo, paste, lifecycle and more).

## Keyboard shortcuts

`editor.addShortcut(pattern, description, cmdOrFn, scope?)` binds a shortcut.
`meta` maps to Cmd on macOS and Ctrl on Windows; `access` maps to Ctrl+Option /
Shift+Alt. The action can be a command name or a function.

```js
editor.addShortcut('meta+shift+e', 'Insert signature', () => {
  editor.insertContent('<p>— Sent from my editor</p>');
});
```

## Undo / redo

`editor.undoManager` manages history levels.

- `undo()` / `redo()` — step through history.
- `hasUndo()` / `hasRedo()` — gate UI buttons.
- `add()` — manually snapshot a level after a programmatic mutation.
- `clear()` — drop all levels; `reset()` clears then adds a fresh baseline.
- `transact(fn)` — run a mutator as **one** undo level. Inner `execCommand` /
  `insertContent` calls collapse into a single step:
  ```js
  editor.undoManager.transact(() => {
    editor.execCommand('mceInsertContent', false, '<h2>Title</h2>');
    editor.insertContent('<p>Body</p>');
  });
  ```

`ignore(fn)` runs a mutation with no undo level at all. See
[references/undo-manager.md](references/undo-manager.md).

## Formatter

`editor.formatter` applies named formats (the engine behind Bold, font size,
custom styles) — more reliable than raw browser commands.

- `register(name, format)` — define a format (inline tag, styles, classes…).
- `apply(name, vars?, node?)` / `remove(...)` / `toggle(...)` — apply, remove or
  flip a format on the selection or a node.
- `match(name, vars?, node?)` — `true` if the selection/node has the format.

```js
editor.formatter.register('highlight', { inline: 'span', styles: { backgroundColor: '#ff0' } });
editor.formatter.toggle('highlight');
```

See [references/formatter.md](references/formatter.md) for the full registry and
matching API.

## Rules of thumb

- Reach content through `getContent()` / `setContent()` / `insertContent()` —
  never read or write the editor's DOM body or the underlying textarea directly.
- Call `getContent()` (or `save()`) at submit time; don't try to keep a shadow
  copy in sync.
- Prefer `change` for "content was committed" logic and `input` for keystroke-level
  feedback.
- Use commands and the formatter for mutations rather than hand-built DOM edits,
  so undo, events and cleanup all work.
- Wrap multi-step programmatic edits in `undoManager.transact()` so they undo as
  one unit.
- Verify a command name with `queryCommandSupported` before relying on it.
