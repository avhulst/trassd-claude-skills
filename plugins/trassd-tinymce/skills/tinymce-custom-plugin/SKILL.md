---
name: tinymce-custom-plugin
description: Authoring a custom TinyMCE plugin and its UI. Use when writing a TinyMCE plugin, adding a custom toolbar button, menu item, context toolbar/menu, or autocompleter, or extending the editor with new commands, shortcuts, or icons via tinymce.PluginManager.add and editor.ui.registry.
---

# Authoring a custom TinyMCE plugin

A custom plugin is a function registered with `tinymce.PluginManager.add`. It
receives the `editor` instance and the plugin's base `url`, registers UI through
`editor.ui.registry.*`, wires up commands/shortcuts, and (optionally) returns
plugin metadata. The plugin only runs when its name is listed in the `plugins`
init option, and its buttons/menu items only appear when their **registered
names** are referenced in the `toolbar` / `menu` options.

## Plugin skeleton

Register the plugin **before** `tinymce.init` runs (it is invoked by the editor
during init, once per editor instance):

```js
tinymce.PluginManager.add('myplugin', (editor, url) => {
  // 1. commands the UI will call
  editor.addCommand('mceMyAction', () => {
    editor.insertContent('&nbsp;<strong>Hello!</strong>&nbsp;');
  });

  // 2. UI registered against this editor instance
  editor.ui.registry.addButton('myplugin', {
    text: 'My button',
    tooltip: 'Insert hello',
    onAction: () => editor.execCommand('mceMyAction')
  });

  // 3. (optional) keyboard shortcut
  editor.addShortcut('meta+shift+h', 'Insert hello', 'mceMyAction');

  // 4. (optional) metadata shown in the Help dialog / about box
  return {
    getMetadata: () => ({ name: 'My Plugin', url: 'https://example.com/myplugin' })
  };
});
```

Then enable it and surface its button:

```js
tinymce.init({
  selector: 'textarea',
  plugins: 'myplugin',          // runs the registered function
  toolbar: 'myplugin'           // shows the button registered as 'myplugin'
});
```

Rules of thumb:

- **Register via `PluginManager.add` before `tinymce.init`.** The name passed to
  `add` is the name you put in the `plugins` option.
- **A button/menu item only shows if its registered name is referenced** in
  `toolbar` (buttons) or `menu`/menu-button items (menu items). Registering it is
  not enough.
- **Put behaviour in a command** (`editor.addCommand`) and have `onAction` call
  `editor.execCommand(...)`. That makes the same action reusable from shortcuts,
  menu items, and external code.
- **Return a teardown from `onSetup`** (see below) — do not leak listeners.
- The plugin function is called **per editor instance**; everything registered on
  `editor`/`editor.ui.registry` is scoped to that instance.

## Common toolbar / menu registrations

All registry methods take `(name, spec)`. The most common specs:

```js
// Basic toolbar button
editor.ui.registry.addButton('myplugin', {
  icon: 'bold', tooltip: 'Do thing',
  onAction: () => editor.execCommand('mceMyAction')
});

// Toggle button — reflect state via the api passed to onSetup
editor.ui.registry.addToggleButton('myfmt', {
  text: 'Highlight',
  onAction: (api) => editor.execCommand('mceToggleFormat', false, 'highlight'),
  onSetup: (api) => {
    const update = () => api.setActive(editor.formatter.match('highlight'));
    editor.on('NodeChange', update);
    return () => editor.off('NodeChange', update); // teardown
  }
});

// Menu button — opens a menu built from menu items
editor.ui.registry.addMenuButton('myinsert', {
  text: 'Insert', tooltip: 'Insert snippet',
  fetch: (callback) => callback([
    { type: 'menuitem', text: 'Date', onAction: () => editor.execCommand('mceMyDate') },
    { type: 'menuitem', text: 'Time', onAction: () => editor.execCommand('mceMyTime') }
  ])
});

// Basic menu item (place its name in the `menu` option to show it)
editor.ui.registry.addMenuItem('myitem', {
  text: 'My item', icon: 'image',
  onAction: () => editor.execCommand('mceMyAction')
});
```

Shared spec fields: `text`, `icon` (registered icon name), `tooltip`,
`onAction`, and `onSetup(api) => teardownFn`. `onSetup` runs when the control is
rendered; return a function to unbind any listeners you added.

## Commands and shortcuts

```js
editor.addCommand('mceMyAction', (ui, value) => { /* do work */ });

// String form: pattern, description, command name to execCommand
editor.addShortcut('meta+shift+h', 'Insert hello', 'mceMyAction');
// Function form
editor.addShortcut('ctrl+alt+9', 'Custom', () => editor.insertContent('!'));
```

Shortcut pattern modifiers (from the Shortcuts API): `ctrl`, `alt`, `shift`,
`meta` ("meta" = Command on macOS, Ctrl on Windows), and `access`
(Control+Option on macOS, Shift+Alt on Windows). Combine with `+`, e.g.
`meta+access+c`. Function-key (`f1`..`f12`) and numeric keycodes
(e.g. `ctrl+219`) are also accepted.

## Full registry catalog and bigger samples

The complete list of `editor.ui.registry` methods (split buttons, nested/toggle
menu items, context toolbars/menus, autocompleters, icons, sidebars, views) with
longer examples lives in the references:

- [references/ui-registry.md](references/ui-registry.md) — every registry method, its spec, and when to use it
- [references/examples.md](references/examples.md) — fuller plugin samples (split button, context toolbar, autocompleter, custom icon, sidebar)
