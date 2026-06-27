---
name: tinymce-bundled-plugins
description: How to enable and configure TinyMCE's bundled open-source plugins via the `plugins` option and place their toolbar/menu items, plus the `PluginManager` API. Use when adding a plugin to the `plugins` list, wiring up its buttons in `toolbar`/`menubar`, or configuring a bundled plugin's behavior (link, image, table, lists, etc.).
---

# TinyMCE bundled plugins

TinyMCE ships a set of open-source plugins inside the core package. You do not
install them separately — you **activate** them by naming them in the `plugins`
option. Activating a plugin usually registers commands plus toolbar buttons and
menu items, which you then **place** by listing those item names in `toolbar`
and/or `menubar`.

## Enabling a plugin: the `plugins` ↔ `toolbar` relationship

`plugins` is a space-separated string (an array also works) of plugin names.
Naming a plugin runs its setup, but most plugins do **not** show any UI until
you reference the controls they register in `toolbar` / `menubar`.

```js
tinymce.init({
  selector: 'textarea',
  // 1. Activate the plugins
  plugins: 'lists advlist link image table code',
  // 2. Place the controls they register (button names)
  toolbar: 'bold italic | bullist numlist | link image | table | code',
  // 3. Menu items they register are exposed via menus
  menubar: 'edit insert format table tools',
});
```

Key points:

- A plugin in `plugins` but absent from `toolbar`/`menubar` is active (its
  commands/behaviours run) but has no visible UI.
- A control name in `toolbar` whose plugin is not in `plugins` simply does not
  render — there is no error, the button is dropped.
- Order in `plugins` does not matter; order in `toolbar` is the visible order.
- Use `|` in `toolbar` to add separators between button groups.

## The PluginManager API

`tinymce.PluginManager` is the registry every bundled plugin registers itself
with. You normally only touch it when authoring or inspecting plugins.

- `tinymce.PluginManager.add(name, setupFn)` — register a plugin. `setupFn`
  receives `(editor, url)` and is the plugin's `init`. This is exactly how the
  bundled plugins (e.g. `'link'`, `'image'`, `'table'`, `'lists'`) register.
- `tinymce.PluginManager.get(name)` — look up a registered plugin (or
  `undefined`).
- `tinymce.PluginManager.requireLangPack(name, languages)` — declare which
  language packs a plugin needs so translations load.

```js
tinymce.PluginManager.add('mycustom', (editor, url) => {
  editor.ui.registry.addButton('mybtn', { text: 'Hi', onAction: () => {} });
  return { getMetadata: () => ({ name: 'My Custom', url: 'https://example.com' }) };
});
```

A plugin's setup may return a `Plugin` object (with an optional `getMetadata`
and any custom API methods); `lists`, for example, returns an API consumable by
other plugins. Returning nothing is also valid.

## Bundled plugin index

These are the plugins bundled with the core package — use any of these names in
`plugins`. Grouped by purpose:

**Formatting & lists**
- `lists` — proper semantic bullet/numbered lists (`bullist`, `numlist`).
- `advlist` — adds style options (e.g. disc/circle, decimal/roman) to the list buttons.

**Inserting content**
- `link` — insert/edit hyperlinks (`link`, `unlink`, `openlink`).
- `image` — insert/edit images, optional captions and upload tab.
- `media` — embed video/audio/iframe (e.g. YouTube) media.
- `table` — create and edit tables with row/column/cell tools.
- `charmap` — insert special characters from a picker.
- `emoticons` — insert emoji.
- `codesample` — insert syntax-highlighted code blocks.
- `anchor` — insert named anchors (bookmarks) to link to.
- `insertdatetime` — insert the current date/time.
- `pagebreak` — insert a page-break marker.
- `nonbreaking` — insert non-breaking spaces.

**Editing aids**
- `searchreplace` — find and replace text.
- `wordcount` — live word/character count.
- `autosave` — restore unsaved content after an accidental close.
- `autolink` — turn typed URLs/emails into links automatically.
- `visualblocks` — toggle visible block-element outlines.
- `visualchars` — toggle visible invisible characters (e.g. nbsp).
- `directionality` — left-to-right / right-to-left controls (`ltr`, `rtl`).

**View / UX**
- `fullscreen` — toggle full-screen editing.
- `preview` — preview content in a dialog.
- `code` — edit the raw HTML source in a dialog.
- `help` — show a help dialog (shortcuts, plugins).
- `quickbars` — contextual inline toolbars for selection, insert, and images.
- `importcss` — pull class names from your stylesheets into the Formats menu.
- `accordion` — insert collapsible accordion sections.
- `autoresize` — grow/shrink the editor height to fit content.
- `save` — a save button that submits the form / runs a save handler.

## Common control & menu names

Plugins register these named controls (place them in `toolbar`/`menubar`):

- lists → `bullist`, `numlist`
- link → `link`, `unlink`, `openlink`
- image → `image`
- table → `table` (button opens a grid; also adds a `Table` menu)
- code → `code`; preview → `preview`; fullscreen → `fullscreen`
- searchreplace → `searchreplace`; charmap → `charmap`; emoticons → `emoticons`
- codesample → `codesample`; anchor → `anchor`; visualblocks → `visualblocks`
- directionality → `ltr`, `rtl`; insertdatetime → `insertdatetime`
- pagebreak → `pagebreak`; nonbreaking → `nonbreaking`

## Per-plugin options

For the configuration options of the most-used plugins (link, image, table,
lists/advlist) see [references/plugin-options.md](references/plugin-options.md).

## Checklist

- Add the plugin name to `plugins`.
- Add its button(s) to `toolbar` and/or rely on its menu items via `menubar`.
- For plugins with options (link, image, table, advlist), set the relevant
  options in the same `init` call.
- Don't reference a button whose plugin you didn't enable — it silently drops.
