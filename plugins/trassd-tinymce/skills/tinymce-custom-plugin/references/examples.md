# Fuller custom-plugin samples

All examples assume they are inside the function passed to
`tinymce.PluginManager.add('myplugin', (editor, url) => { ... })`, and that
`'myplugin'` is added to the `plugins` init option (and the relevant button name
to `toolbar`).

## Split button

A primary action plus a dropdown of variants.

```js
editor.ui.registry.addSplitButton('myformats', {
  text: 'Insert',
  tooltip: 'Insert snippet',
  onAction: () => editor.insertContent('<hr />'),         // primary click
  onItemAction: (api, value) => editor.insertContent(value),
  fetch: (callback) => callback([
    { type: 'choiceitem', text: 'Horizontal rule', value: '<hr />' },
    { type: 'choiceitem', text: 'Line break',      value: '<br />' }
  ])
});
```

## Toggle button reflecting editor state

```js
editor.addCommand('mceToggleHighlight', () => {
  editor.formatter.toggle('highlight');
});

editor.ui.registry.addToggleButton('highlight', {
  icon: 'highlight-bg-color',
  tooltip: 'Highlight',
  onAction: () => editor.execCommand('mceToggleHighlight'),
  onSetup: (api) => {
    const update = () => api.setActive(editor.formatter.match('highlight'));
    editor.on('NodeChange', update);
    return () => editor.off('NodeChange', update); // teardown
  }
});
```

## Nested and toggle menu items

```js
editor.ui.registry.addNestedMenuItem('insertstuff', {
  text: 'Insert stuff',
  getSubmenuItems: () => [
    { type: 'menuitem', text: 'Date', onAction: () => editor.insertContent(new Date().toDateString()) },
    { type: 'menuitem', text: 'Rule', onAction: () => editor.insertContent('<hr/>') }
  ]
});

editor.ui.registry.addToggleMenuItem('togglecmt', {
  text: 'Show comments',
  onAction: (api) => { /* toggle something */ },
  onSetup: (api) => {
    api.setActive(false);
    return () => {};
  }
});
```

## Context toolbar (appears on a matched element)

```js
editor.ui.registry.addButton('editlink', {
  icon: 'link',
  tooltip: 'Edit link',
  onAction: () => editor.execCommand('mceLink')
});

editor.ui.registry.addContextToolbar('linktoolbar', {
  predicate: (node) => node.nodeName.toLowerCase() === 'a',
  items: 'editlink',          // names of registered buttons
  position: 'node',
  scope: 'node'
});
```

## Context menu section

```js
editor.ui.registry.addContextMenu('mycontext', {
  update: (element) =>
    element.nodeName.toLowerCase() === 'img' ? 'editimage' : ''
});
```

## Autocompleter (trigger on a typed pattern)

```js
editor.ui.registry.addAutocompleter('mentions', {
  trigger: '@',
  minChars: 1,
  columns: 1,
  fetch: (pattern) => new Promise((resolve) => {
    const users = ['ann', 'bob', 'cleo'].filter((u) => u.indexOf(pattern) === 0);
    resolve(users.map((u) => ({ type: 'autocompleteitem', value: u, text: '@' + u })));
  }),
  onAction: (autocompleteApi, range, value) => {
    editor.selection.setRng(range);
    editor.insertContent('@' + value);
    autocompleteApi.hide();
  }
});
```

## Custom SVG icon

```js
// Grounded in the addIcon JSDoc example:
editor.ui.registry.addIcon(
  'triangleUp',
  '<svg height="24" width="24"><path d="M12 0 L24 24 L0 24 Z" /></svg>'
);

editor.ui.registry.addButton('triangle', {
  icon: 'triangleUp',           // reference the registered icon name
  onAction: () => editor.insertContent('▲')
});
```

## Sidebar

Registering a sidebar also creates a toggle button with the same name plus a
`ToggleSidebar` command and event.

```js
editor.ui.registry.addSidebar('mysidebar', {
  tooltip: 'My sidebar',
  icon: 'comment',
  onSetup: (api) => {
    api.element().innerHTML = '<p>Side panel content</p>';
    return () => {};
  },
  onShow: (api) => { /* ... */ },
  onHide: (api) => { /* ... */ }
});
// toggle programmatically:
editor.execCommand('ToggleSidebar', false, 'mysidebar');
```

## Commands and shortcuts together

```js
editor.addCommand('mceInsertHello', () => {
  editor.execCommand('mceInsertContent', false, 'Hello, World!');
});

// "meta" = Command on macOS, Ctrl on Windows
editor.addShortcut('meta+shift+h', 'Insert greeting', 'mceInsertHello');

// "access" = Control+Option on macOS, Shift+Alt on Windows; function form:
editor.addShortcut('access+g', 'Greet', () => editor.execCommand('mceInsertHello'));
```
