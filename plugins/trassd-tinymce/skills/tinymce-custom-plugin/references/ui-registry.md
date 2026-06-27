# `editor.ui.registry` method catalog

Every method is called as `editor.ui.registry.<method>(name, spec)`. The `name`
is unique and is what you reference from the `toolbar` / `menu` init options (for
buttons and menu items) or what the editor uses internally (icons, contexts,
autocompleters, sidebars, views). Register everything inside the plugin function
passed to `tinymce.PluginManager.add`.

## Toolbar buttons

| Method | Use for |
| --- | --- |
| `addButton(name, spec)` | A basic toolbar button that runs an action when clicked or activated by keyboard. Reference `name` in `toolbar`. |
| `addToggleButton(name, spec)` | A button with an on/off state; set state via the `api` in `onAction`/`onSetup` (`api.setActive(bool)`). |
| `addMenuButton(name, spec)` | A button that opens a menu; populate via `fetch(callback)` returning menu items (made with `addMenuItem`/`addNestedMenuItem`/`addToggleMenuItem`). |
| `addSplitButton(name, spec)` | A button with a primary action plus a dropdown of choices. |
| `addGroupToolbarButton(name, spec)` | A button that opens a floating toolbar. **Only works in floating toolbar mode.** |

## Menu items

| Method | Use for |
| --- | --- |
| `addMenuItem(name, spec)` | A basic menu item that runs an action. Reference `name` in the `menu` option. |
| `addNestedMenuItem(name, spec)` | A menu item revealing a submenu; populate via `getSubmenuItems()`. |
| `addToggleMenuItem(name, spec)` | A menu item that toggles, showing a tick to represent state. |

## Contextual UI

| Method | Use for |
| --- | --- |
| `addContextToolbar(name, spec)` | A toolbar that appears only when a content `predicate` matches (e.g. cursor on an image). |
| `addContextMenu(name, spec)` | A context-menu section shown when a `predicate` matches (e.g. cursor inside a table). |
| `addContextForm(name, spec)` | An inline input form shown when a `predicate` matches (e.g. quick-edit a link URL). |
| `addAutocompleter(name, spec)` | Triggers when a configured string pattern (e.g. `:`, `@`) is typed; used by emoticons/charmap. |

## Icons, contexts, panels

| Method | Use for |
| --- | --- |
| `addIcon(name, svgData)` | Register an SVG icon string; the `name` can then be used as any control's `icon`. Scoped to that editor instance. |
| `addContext(name, pred)` | Register a named predicate; controls with a matching `context` property enable/disable based on it. |
| `addSidebar(name, spec)` | A toggleable panel attached to the right of the editor. Registering it also creates a toggle button of the same name and a `ToggleSidebar` command/event. |
| `addView(name, spec)` | A full-screen alternate view next to the main one; toggled with the `ToggleView` command (which can also be queried for state). |

## Common spec fields

Most button/menu specs share these:

- `text` — visible label.
- `icon` — name of a registered icon (built-in or from `addIcon`).
- `tooltip` — hover/access description.
- `onAction(api)` — invoked on click / keyboard activation. Prefer calling
  `editor.execCommand('myCommand')` here.
- `onSetup(api) => teardownFn` — invoked when the control is rendered. Bind any
  editor listeners you need (e.g. `editor.on('NodeChange', ...)`) and **return a
  function that unbinds them**. The returned function runs when the control is
  removed.

State helpers available on the `api` object vary by control:
- Toggle button / toggle menu item: `api.setActive(boolean)`, `api.isActive()`.
- Most controls: `api.setEnabled(boolean)`, `api.isEnabled()`.

Menu-population callbacks:
- Menu button: `fetch: (callback) => callback(items)`.
- Nested menu item: `getSubmenuItems: () => items`.
- Split button: `fetch: (callback) => callback(choiceItems)`.

> `getAll()` exists on the registry but is documented as internal and may not be
> supported in future revisions — do not rely on it.
