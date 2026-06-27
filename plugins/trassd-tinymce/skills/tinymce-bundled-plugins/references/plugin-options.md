# Per-plugin configuration options

Options for the most-configured bundled plugins. Set them in the same
`tinymce.init({ ... })` call that lists the plugin in `plugins`. Every name
below is registered by the corresponding plugin.

## link

Activate with `link` in `plugins`. Registers `link`, `unlink`, `openlink`.

| Option | Purpose |
| --- | --- |
| `link_default_target` | Default `target` applied to new links (e.g. `'_blank'`). |
| `link_default_protocol` | Protocol used when a URL has none (e.g. `'https'`). |
| `link_assume_external_targets` | Prompt/assume `http(s)`/`mailto` for external-looking URLs. |
| `link_target_list` | Custom list of target choices in the dialog (or `false` to hide). |
| `link_rel_list` | Selectable `rel` values for the link. |
| `link_class_list` | Selectable CSS classes for the link. |
| `link_title` | Show/hide the Title field in the dialog. |
| `link_list` | Predefined list of links (array or URL/callback) for the dialog. |
| `link_context_toolbar` | Enable the inline context toolbar for editing a link. |
| `link_quicklink` | Use the simplified quick-link UI. |
| `allow_unsafe_link_target` | Allow `target` without adding `rel="noopener"`. |
| `link_default_protocol` | Protocol prepended to protocol-less URLs. |

```js
tinymce.init({
  plugins: 'link',
  toolbar: 'link unlink',
  link_default_target: '_blank',
  link_default_protocol: 'https',
  link_context_toolbar: true,
});
```

## image

Activate with `image` in `plugins`. Registers `image`.

| Option | Purpose |
| --- | --- |
| `image_caption` | Allow figure/figcaption captions on images. |
| `image_advtab` | Show the Advanced tab in the image dialog. |
| `image_dimensions` | Show width/height fields (set `false` to hide). |
| `image_title` | Show the Title field. |
| `image_description` | Show the alt/description field. |
| `image_class_list` | Selectable CSS classes for the image. |
| `image_list` | Predefined list of images for the dialog (array/URL/callback). |
| `image_prepend_url` | Base URL prepended to inserted `src` values. |
| `image_uploadtab` | Show the Upload tab in the dialog. |
| `a11y_advanced_options` | Show advanced accessibility (alt/decorative) options. |

```js
tinymce.init({
  plugins: 'image',
  toolbar: 'image',
  image_caption: true,
  image_advtab: true,
  image_dimensions: false,
});
```

## table

Activate with `table` in `plugins`. Registers the `table` button and a Table menu.

| Option | Purpose |
| --- | --- |
| `table_toolbar` | Controls shown on the table context toolbar (or `''` to hide). |
| `table_grid` | Use the visual grid picker for inserting tables. |
| `table_advtab` | Show the Advanced tab in the table dialog. |
| `table_cell_advtab` | Advanced tab for cell properties. |
| `table_row_advtab` | Advanced tab for row properties. |
| `table_class_list` | Selectable CSS classes for the table. |
| `table_cell_class_list` | Selectable classes for cells. |
| `table_row_class_list` | Selectable classes for rows. |
| `table_default_attributes` | Attributes applied to new tables (e.g. `border`). |
| `table_default_styles` | Styles applied to new tables (e.g. `width`). |
| `table_sizing_mode` | How sizes are applied: e.g. `'fixed'`, `'relative'`, `'responsive'`. |
| `table_style_by_css` | Style tables via CSS rather than HTML attributes. |
| `table_appearance_options` | Show legacy appearance options (cellpadding, etc.). |
| `table_border_styles` / `table_border_widths` / `table_border_color_map` | Choices for borders. |
| `table_background_color_map` | Background color swatches for cells. |

```js
tinymce.init({
  plugins: 'table',
  toolbar: 'table',
  menubar: 'table',
  table_sizing_mode: 'responsive',
  table_default_attributes: { border: '1' },
  table_class_list: [
    { title: 'None', value: '' },
    { title: 'Striped', value: 'striped' },
  ],
});
```

## lists & advlist

`lists` provides the semantic list behaviour and the `bullist` / `numlist`
buttons. `advlist` augments those buttons with a dropdown of list styles, so
enable both for rich list controls.

| Option (advlist) | Purpose |
| --- | --- |
| `advlist_bullet_styles` | Comma-separated bullet styles offered (e.g. `'default,circle,disc,square'`). |
| `advlist_number_styles` | Numbered styles offered (e.g. `'default,lower-alpha,lower-roman,upper-alpha,upper-roman'`). |

```js
tinymce.init({
  plugins: 'lists advlist',
  toolbar: 'bullist numlist',
  advlist_bullet_styles: 'disc,circle,square',
  advlist_number_styles: 'decimal,lower-alpha,lower-roman',
});
```

> `lists` must be enabled for `advlist` to have buttons to enhance.
