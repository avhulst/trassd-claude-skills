# Options catalog

A broader list of registered TinyMCE options grouped by purpose. Types and defaults are
taken from the editor's option registration. All are passed inside `tinymce.init({...})`
and read back with `editor.options.get('name')`. Only options whose plugin/model is loaded
are available.

## Resources & loading

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `base_url` | string | — | Base URL for TinyMCE's own resources. |
| `suffix` | string | — | Resource filename suffix, e.g. `.min`. |
| `cache_suffix` | string | — | Appended to resource URLs for cache busting. |
| `referrer_policy` | string | `''` | `referrerpolicy` for injected resources. |
| `crossorigin` | function | — | `(url, type) => 'anonymous' | 'use-credentials' | undefined` for injected resources. |
| `language` | string | `'en'` | UI language code. |
| `language_url` | string | `''` | URL of the language pack. |
| `language_load` | boolean | `true` | Whether to load the language pack. |
| `theme` | string \| false \| function | `'silver'` | UI theme; `false` for none. |
| `theme_url` | string | — | Custom theme URL. |
| `model` | string | `'dom'` | Editor model. |
| `model_url` | string | — | Custom model URL. |
| `external_plugins` | object | — | `{ name: url }` map of plugins loaded from explicit URLs. |
| `forced_plugins` | string[] | — | Plugins always loaded (used by integrations). |
| `icons` | string | `''` | Icon pack name. |
| `icons_url` | string | `''` | Icon pack URL. |

## Content styling & body

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `content_css` | boolean \| string \| string[] | `['default']` (`[]` inline) | Stylesheets inside the editable area. |
| `content_style` | string | — | Raw CSS injected into the content area. |
| `content_css_cors` | boolean | `false` | Add `crossorigin` to content stylesheet links. |
| `content_language` | string | — | `lang` for the content body. |
| `directionality` | string | `'rtl'` if RTL locale | `'ltr'` or `'rtl'`. |
| `body_id` | string | `'tinymce'` | `id` of the content body. |
| `body_class` | string | `''` | `class` of the content body. |
| `font_css` | string \| string[] | `[]` | Extra font stylesheets. |
| `iframe_attrs` | object | `{}` | Attributes set on the editor iframe. |
| `iframe_aria_text` | string | `'Rich Text Area'` | ARIA label for the iframe. |

## Editing behaviour

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `readonly` | boolean | `false` | Non-editable content. |
| `disabled` | boolean | `false` | Disabled state (fires a disabled-state change). |
| `editable_root` | boolean | `true` | Whether the root is editable. |
| `auto_focus` | string \| true | — | Editor id to focus (or `true`) after init. |
| `forced_root_block` | string (non-empty) | `'p'` | Block element wrapping top-level content. |
| `forced_root_block_attrs` | object | `{}` | Attributes for the forced root block. |
| `newline_behavior` | `'block'`/`'linebreak'`/`'invert'`/`'default'` | `'default'` | Enter-key behaviour. |
| `br_in_pre` | boolean | `true` | Use `<br>` inside `<pre>`. |
| `indent` | boolean | `true` | Pretty-print/indent output. |
| `indentation` | string | `'40px'` | Indent step for the indent command. |
| `indent_use_margin` | boolean | `false` | Indent via margin instead of padding. |
| `object_resizing` | boolean \| string | `true` (off on touch) | Selectors of resizable objects. |
| `resize_img_proportional` | boolean | `true` | Keep aspect ratio when resizing images. |
| `keep_styles` | boolean | `true` | Carry styles onto new blocks on Enter. |
| `browser_spellcheck` | boolean | `false` | Native browser spellcheck. |
| `custom_undo_redo_levels` | number | `0` | Max undo levels (`0` = unlimited). |
| `highlight_on_focus` | boolean | `true` | Highlight editor on focus. |
| `placeholder` | string | source element's `placeholder` | Empty-state hint. |

## Serialization, schema & validation

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `schema` | string | `'html5'` | HTML schema. |
| `element_format` | string | `'html'` | `'html'` or `'xhtml'` output. |
| `entity_encoding` | string | `'named'` | Entity encoding strategy. |
| `entities` | string | — | Custom entity map. |
| `encoding` | string | — | `'xml'` to XML-encode submitted content. |
| `valid_elements` | string | — | Allowed elements/attributes. |
| `extended_valid_elements` | string | — | Additional allowed elements. |
| `invalid_elements` | string | — | Disallowed elements. |
| `valid_children` | string | — | Allowed parent/child relationships. |
| `valid_classes` | string \| object | — | Allowed classes. |
| `valid_styles` | string \| object | — | Allowed inline styles. |
| `invalid_styles` | string \| object | — | Disallowed inline styles. |
| `custom_elements` | string \| object | — | Register custom elements. |
| `verify_html` | boolean | `true` | Validate HTML against the schema. |
| `fix_list_elements` | boolean | `false` | Repair malformed lists. |
| `protect` | array (RegExp[]) | — | Regexes for content protected from processing. |
| `convert_fonts_to_spans` | boolean (deprecated) | `true` | Convert `<font>` to spans. |
| `inline_styles` | boolean (deprecated) | `true` | Use inline styles for formatting. |

## URLs

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `convert_urls` | boolean | `true` | Convert URLs on save. |
| `relative_urls` | boolean | `true` | Emit relative URLs. |
| `remove_script_host` | boolean | `true` | Strip host from same-origin URLs. |
| `document_base_url` | string | document base | Base for URL conversion. |
| `url_converter` | function | editor's converter | Custom URL converter. |
| `url_converter_scope` | object | editor | `this` for the converter. |
| `urlconverter_callback` | function | — | Per-URL conversion callback. |

## Paste

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `paste_as_text` | boolean | `false` | Paste as plain text. |
| `paste_data_images` | boolean | `true` | Allow pasted data-URI images. |
| `paste_block_drop` | boolean | `false` | Block drag-and-drop pasting. |
| `paste_merge_formats` | boolean | `true` | Merge adjacent identical formats on paste. |
| `paste_tab_spaces` | number | `4` | Spaces a tab becomes when pasting. |
| `paste_webkit_styles` | string | `'none'` | WebKit styles to retain. |
| `paste_remove_styles_if_webkit` | boolean | `true` | Strip WebKit styles on paste. |
| `smart_paste` | boolean | `true` | Smart link/image paste detection. |
| `paste_preprocess` | function | — | `(editor, args)` before paste processing. |
| `paste_postprocess` | function | — | `(editor, args)` after paste processing. |

## Images & uploads

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `automatic_uploads` | boolean | `true` | Auto-upload local/blob images. |
| `images_upload_url` | string | `''` | Endpoint receiving uploads. |
| `images_upload_base_path` | string | `''` | Base path prepended to upload URLs. |
| `images_upload_credentials` | boolean | `false` | Send cookies with uploads. |
| `images_upload_handler` | function | — | Custom upload handler. |
| `images_reuse_filename` | boolean | `false` | Reuse the original filename. |
| `images_replace_blob_uris` | boolean | `true` | Replace blob URIs after upload. |
| `images_file_types` | string | `jpeg,jpg,jpe,jfi,jif,jfif,png,gif,bmp,webp` | Accepted image extensions. |

## Security & sanitization

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `xss_sanitization` | boolean | `true` | Sanitize content against XSS. Keep on. |
| `sandbox_iframes` | boolean | `true` | Sandbox embedded iframes. |
| `sandbox_iframes_exclusions` | string[] | common video/embed hosts | Hosts exempt from sandboxing. |
| `convert_unsafe_embeds` | boolean | `true` | Convert unsafe `<embed>`/`<object>`. |
| `content_security_policy` | string | `''` | CSP for the editor iframe. |
| `allow_script_urls` | boolean | `false` | Permit `javascript:` URLs. Keep off. |
| `allow_html_data_urls` | boolean | `false` | Permit HTML `data:` URLs. |
| `allow_svg_data_urls` | boolean | — | Permit SVG `data:` URLs. |
| `allow_conditional_comments` | boolean | `false` | Keep IE conditional comments. |
| `allow_unsafe_link_target` | boolean | `false` | Allow `target` without `rel=noopener`. |

## Form integration & callbacks

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `setup` | function | — | `(editor) => {...}` run during init. |
| `init_instance_callback` | function | — | `(editor) => {...}` after init completes. |
| `hidden_input` | boolean | `true` | Create a hidden input for form submit. |
| `submit_patch` | boolean | `true` | Patch the form's submit to save the editor. |
| `add_form_submit_trigger` | boolean | `true` | Save on form submit. |
| `add_unload_trigger` | boolean | `true` | Save on page unload. |

## Misc

| Option | Type | Default | Notes |
| --- | --- | --- | --- |
| `license_key` | string | — | License key (e.g. `'gpl'`). |
| `api_key` | string | — | Tiny Cloud API key. |
| `text_patterns` | object[] \| false | markdown-style defaults | Auto-format trigger patterns. |
| `text_patterns_lookup` | function | `() => []` | Dynamic pattern lookup. |
| `list_max_depth` | number | — | Max list nesting depth. |
| `lists_indent_on_tab` | boolean | `true` | Tab indents list items. |
| `details_initial_state` | `'inherited'`/`'collapsed'`/`'expanded'` | `'inherited'` | Initial `<details>` state. |
| `details_serialized_state` | same | `'inherited'` | Serialized `<details>` state. |
| `deprecation_warnings` | boolean | `true` | Log deprecation warnings. |
