# Toolbar, menu & layout recipes

All examples are usage (`tinymce.init({...})`). Item names in `toolbar`/`menu`/`menubar`
are space-separated; `|` is a group separator. A name only renders if core or an enabled
plugin provides it.

## Single-row toolbar with groups

```js
tinymce.init({
  selector: '#editor',
  plugins: 'lists link',
  toolbar: 'undo redo | styles | bold italic | bullist numlist | link'
});
```

## Multiple toolbar rows

Array form — one string per row:

```js
tinymce.init({
  selector: '#editor',
  plugins: 'lists link image table',
  toolbar: [
    'undo redo | bold italic underline',
    'bullist numlist | link image | table'
  ]
});
```

Numbered form (`toolbar1` … `toolbar9`) is equivalent:

```js
tinymce.init({
  selector: '#editor',
  toolbar1: 'undo redo | bold italic underline',
  toolbar2: 'bullist numlist | link image'
});
```

## Named toolbar groups

`toolbar` also accepts an array of group objects (`{ name?, label?, items: string[] }`):

```js
tinymce.init({
  selector: '#editor',
  toolbar: [
    { name: 'history', items: ['undo', 'redo'] },
    { name: 'formatting', items: ['bold', 'italic', 'underline'] },
    { name: 'lists', items: ['bullist', 'numlist'] }
  ]
});
```

## Toolbar overflow & placement

```js
tinymce.init({
  selector: '#editor',
  toolbar_mode: 'sliding',     // 'floating' | 'sliding' | 'scrolling' | 'wrap'
  toolbar_location: 'top',     // 'top' | 'bottom' | 'auto'
  toolbar_sticky: true,
  toolbar_sticky_offset: 64
});
```

- `floating` shows an overflow (`…`) button with a floating panel.
- `sliding` slides extra buttons into view.
- `scrolling` keeps one row and scrolls horizontally.
- `wrap` wraps buttons onto multiple rows.

## Menu bar & custom menus

```js
tinymce.init({
  selector: '#editor',
  menubar: 'file edit insert format',   // false to hide entirely
  menu: {
    insert: { title: 'Insert', items: 'link image | hr' },
    format: { title: 'Format', items: 'bold italic underline | removeformat' }
  },
  removed_menuitems: 'newdocument'      // drop specific items
});
```

`menu` entries are `{ title, items }` where `items` is a space-separated item string using
the same names as the toolbar.

## Inline editing

```js
tinymce.init({
  selector: '.editable',
  inline: true,                         // edit element in place, no iframe
  toolbar: 'bold italic | link',
  menubar: false
});
```

With `inline: true`, `content_css` defaults to `[]` because content lives in the host page
(style it with your own page CSS).

## Disabling chrome

```js
tinymce.init({
  selector: '#editor',
  menubar: false,
  statusbar: false,
  branding: false,
  resize: false       // false | true (vertical) | 'both'
});
```

## Sizing

```js
tinymce.init({
  selector: '#editor',
  height: 600,         // number (px) or CSS length string
  width: '100%',
  min_height: 300,
  max_height: 800
});
```

## setup callback

`setup(editor)` runs early — register UI and bind events here:

```js
tinymce.init({
  selector: '#editor',
  setup: (editor) => {
    editor.ui.registry.addButton('saveDraft', {
      text: 'Save',
      onAction: () => save(editor.getContent())
    });
    editor.on('init', () => console.log('ready'));
  },
  toolbar: 'saveDraft | bold italic'
});
```

## Reading resolved values at runtime

```js
const ed = tinymce.get('editor');
ed.options.get('toolbar');        // resolved value (set value or default)
ed.options.isSet('height');       // true if explicitly provided
ed.options.set('readonly', true); // change at runtime
ed.options.debug();               // log the raw init config
```
