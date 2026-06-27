# TinyMCE Editor Events

Subscribe/unsubscribe on an editor instance:

```js
editor.on('change', handler);   // subscribe
editor.once('init', handler);   // one-shot
editor.off('change', handler);  // unsubscribe (omit handler to remove all)
editor.dispatch('NodeChange', { /* data */ }); // fire an event
```

Handlers receive an `EditorEvent` wrapping the data below, with
`preventDefault()` / `stopPropagation()` / `isDefaultPrevented()`. Cancelling a
`Before*` event (via `preventDefault()`) aborts the pending action.

## Lifecycle

| Event | Fired when |
| --- | --- |
| `init` | Editor finished initializing and is ready to use. |
| `PreInit` | Before init completes (early hook). |
| `PostRender` | After the editor UI has rendered. |
| `remove` | The editor is being removed. |
| `detach` | The editor is detached. |
| `show` / `hide` | Editor shown / hidden (via `show()` / `hide()`). |
| `dirty` | Editor transitions from clean to dirty (fires once). |
| `SwitchMode` | Editor mode changed — data `{ mode }`. |
| `ScriptsLoaded` | External scripts have loaded. |
| `SkinLoaded` | Skin finished loading. |

Load-error events carrying `{ message }`: `SkinLoadError`, `PluginLoadError`,
`ModelLoadError`, `IconsLoadError`, `ThemeLoadError`, `LanguageLoadError`.

## Content I/O

| Event | Data | Notes |
| --- | --- | --- |
| `BeforeGetContent` | `GetContentArgs` (`+ selection?`) | Cancellable; runs before content is read. |
| `GetContent` | `+ content: string` | Content was read; `content` is mutable. |
| `BeforeSetContent` | `SetContentArgs + content` (`+ selection?`) | Cancellable; mutate `content` to alter what gets set. |
| `SetContent` | `+ content` | Content has been set. |
| `SaveContent` | `GetContentEvent + save: boolean` | Fired by `save()` before writing to the element. |
| `RawSaveContent` | same | Fired by `save()` when format is `raw`. |
| `LoadContent` | `{ load: boolean; element }` | Fired by `load()`. |

Pre/post processing of serialized content: `PreProcess` (`{ node, ...ParserArgs }`),
`PostProcess` (`{ content, ...ParserArgs }`).

## Editing / change tracking

| Event | Data | Use |
| --- | --- | --- |
| `change` | `{ level, lastLevel }` | A committed change (new undo level). Autosave, enable Save. Not per-keystroke. |
| `input` | `InputEvent` | Per input. Live counters / instant feedback. |
| `beforeinput` | `InputEvent` | Before input is applied. |
| `NodeChange` | `{ element, parents, selectionChange?, initial? }` | Selection moved; drive toolbar state. |
| `SelectionChange` | `{}` | Selection changed. |
| `NewBlock` | `{ newBlock }` | A new block element was created. |

## Undo / redo

| Event | Data |
| --- | --- |
| `Undo` / `Redo` | `{ level }` |
| `BeforeAddUndo` / `AddUndo` | `{ level, lastLevel, originalEvent }` |
| `ClearUndos` | `{}` |
| `TypingUndo` | `{}` |

## Commands

| Event | Data |
| --- | --- |
| `BeforeExecCommand` | `{ command, ui, value }` — cancellable. |
| `ExecCommand` | `{ command, ui, value }` |

## Formatting

`FormatApply` / `FormatRemove` — `{ format, vars?, node? }`.
`PreviewFormats` / `AfterPreviewFormats` — `{}`.

## Selection / caret ranges

`GetSelectionRange` `{ range }`, `SetSelectionRange` / `AfterSetSelectionRange`
`{ range, forward }`, `ShowCaret` `{ target, direction, before }`,
`ObjectSelected` / `BeforeObjectSelected` `{ target, targetClone? }`.

## Objects, tables, resize, scroll

| Event | Data |
| --- | --- |
| `ObjectResizeStart` / `ObjectResized` | `{ target, width, height, origin }` |
| `TableModified` | `{ table, structure, style }` |
| `NewRow` | `{ node }` (table row) |
| `NewCell` | `{ node }` (table cell) |
| `ScrollIntoView` / `AfterScrollIntoView` | `{ elm, alignToTop }` |
| `resize` / `scroll` | UI/`UIEvent` |
| `ResizeWindow` / `ScrollWindow` | window UI events |

## Focus & activation

`focus` `{ blurredEditor }`, `blur` `{ focusedEditor }`,
`activate` `{ relatedTarget }`, `deactivate` `{ relatedTarget }`.

## Progress, placeholder, notifications, paste

| Event | Data |
| --- | --- |
| `ProgressState` | `{ state, time? }` (set via `setProgressState`) |
| `AfterProgressState` | `{ state }` |
| `PlaceholderToggle` | `{ state }` |
| `PastePreProcess` | `{ content, internal }` |
| `PastePostProcess` | `{ node, internal }` |
| `PastePlainTextToggle` | `{ state }` |
| `BeforeOpenNotification` | `{ notification }` (spec) |
| `OpenNotification` | `{ notification }` (api) |

## Autocompleter

`AutocompleterStart`, `AutocompleterUpdate`, `AutocompleterEnd`.

## Touch & misc

`tap` / `longpress` (`TouchEvent`), `longpresscancel`, `SetAttrib`.

## EditorManager-level events

Dispatched on `tinymce` (the manager), not a single editor:
`AddEditor` / `RemoveEditor` `{ editor }`, `BeforeUnload` `{ returnValue }`.
