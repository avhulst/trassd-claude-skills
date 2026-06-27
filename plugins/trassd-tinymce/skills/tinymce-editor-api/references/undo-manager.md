# TinyMCE UndoManager

Accessed as `editor.undoManager`. Manages the editor's undo/redo history as a
stack of levels. Prefer these methods over hand-rolled history; they keep the
selection bookmark and `change` events consistent.

## Stepping through history

| Method | Returns | Notes |
| --- | --- | --- |
| `undo()` | undo level or `undefined` | Undo the last action. |
| `redo()` | redo level or `undefined` | Redo the last undone action. |
| `hasUndo()` | `boolean` | Whether any undo levels exist — gate an Undo button. |
| `hasRedo()` | `boolean` | Whether any redo levels exist — gate a Redo button. |

```js
const um = editor.undoManager;
undoBtn.disabled = !um.hasUndo();
redoBtn.disabled = !um.hasRedo();
um.undo();
```

## Managing levels

| Method | Notes |
| --- | --- |
| `add(level?, event?)` | Add a snapshot to the stack; returns the added level or `null` if none was needed. Call after a programmatic mutation you want undoable. |
| `clear()` | Remove all undo levels. |
| `reset()` | Clear all levels, then add a fresh initial level (new baseline). |
| `beforeChange()` | Store a selection bookmark to use for the next undo. |
| `dispatchChange()` | Mark dirty and dispatch a `change` event with the current level. |

`editor.resetContent()` internally calls `undoManager.reset()`, so you usually
don't need to reset history yourself after a content reset.

## Grouping mutations

| Method | Behavior |
| --- | --- |
| `transact(callback)` | Run `callback` and collapse all its changes into **one** undo level. Nested `add`/level logic inside is ignored, so the callback can freely call `execCommand` / `insertContent`. Returns the added level or `null`. |
| `ignore(callback)` | Run `callback` with **no** undo level added at all. |
| `extra(callback1, callback2)` | Apply `callback1` as a hidden extra level, roll it back, then apply `callback2` as the visible change. The hidden level surfaces only on undo. |

```js
// Multiple edits, single undo step:
editor.undoManager.transact(() => {
  editor.execCommand('mceInsertContent', false, '<h2>Section</h2>');
  editor.insertContent('<p>Paragraph</p>');
});

// Mutation that should not be undoable:
editor.undoManager.ignore(() => {
  editor.dom.setAttrib(editor.getBody(), 'data-touched', 'true');
});
```

## `typing` field

`editor.undoManager.typing` is a boolean reflecting whether the user is mid-typing
(so keystrokes coalesce into one level rather than one per key).
