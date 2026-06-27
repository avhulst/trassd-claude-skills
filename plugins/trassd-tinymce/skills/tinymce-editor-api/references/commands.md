# TinyMCE Commands API

Commands are named operations the editor can execute, query, or have custom
handlers registered for. They are the recommended way to mutate content or read
toggle/value state, since they fire the proper events and integrate with undo.

## Executing

```js
editor.execCommand(cmd, ui?, value?, args?): boolean
```

- `cmd` — command name (case-insensitive), e.g. `'Bold'`, `'mceLink'`.
- `ui` — `true` to present a dialog/UI where the command supports one (default `false`).
- `value` — optional value (any type) passed to the command.
- `args` — optional object; supports `{ skip_focus: true }` to avoid focusing the
  editor before executing.
- Returns `true` if a matching command was found and handled, otherwise `false`.

`execCommand` focuses the editor first (except for a small set of internal
commands such as the undo-level commands and `mceFocus`), then dispatches
`BeforeExecCommand` (cancellable) and, on success, `ExecCommand`.

Examples:

```js
editor.execCommand('Bold');
editor.execCommand('mceInsertContent', false, '<em>hello</em>');
editor.execCommand('SelectAll');
editor.execCommand('Bold', false, undefined, { skip_focus: true });
```

`editor.insertContent(html)` is a thin wrapper over
`execCommand('mceInsertContent', false, html)`. `editor.focus(skipFocus?)` wraps
`execCommand('mceFocus', false, skipFocus)`.

## Querying

| Method | Returns | Meaning |
| --- | --- | --- |
| `editor.queryCommandState(cmd)` | `boolean` | Toggle state, e.g. is the selection bold. `false` if unknown or editor hidden/removed. |
| `editor.queryCommandValue(cmd)` | `string` | Current value, e.g. font size; `''` if not found. |
| `editor.queryCommandSupported(cmd)` | `boolean` | Whether an exec handler is registered for the command. |

```js
if (editor.queryCommandState('Bold')) { /* selection is bold */ }
const size = editor.queryCommandValue('FontSize');
if (editor.queryCommandSupported('mceLink')) { /* link plugin available */ }
```

## Registering custom commands

```js
editor.addCommand(name, callback, scope?)
```

The callback is invoked as `callback(ui, value, args)`, with `this` bound to
`scope` (defaults to the editor). Best registered inside `setup`.

```js
editor.addCommand('insertTimestamp', () => {
  editor.insertContent(new Date().toISOString());
});
// later, or from a toolbar button:
editor.execCommand('insertTimestamp');
```

To back a custom toolbar toggle/value, also register handlers:

```js
editor.addQueryStateHandler('insertTimestamp', () => false);
editor.addQueryValueHandler('myFont', () => editor.queryCommandValue('FontName'));
```

- `addQueryStateHandler(name, callback, scope?)` — `callback` returns `boolean`
  (consulted by `queryCommandState`).
- `addQueryValueHandler(name, callback, scope?)` — `callback` returns `string`
  (consulted by `queryCommandValue`).

All three registration methods can override existing commands of the same name.
