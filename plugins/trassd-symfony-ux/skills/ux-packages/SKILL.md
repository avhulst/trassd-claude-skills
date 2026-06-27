---
name: ux-packages
description: PHP-first guide to the prebuilt Symfony UX bridge packages (Chart.js, Cropper.js, Dropzone, Notify, Icons, Map, Translator, CalendarLink). Use when adding Chart.js charts, Cropper.js image cropping, drag-and-drop file uploads, browser notifications, SVG icons, interactive maps, JS-side translations, or "Add to calendar" links to a Symfony template or form.
---

# Symfony UX bridge packages

These bundles each wrap a JavaScript library (or browser API) behind a **PHP
entry point** and a **Twig render call**. The rule of thumb is the same for all
of them: **build the object/configure the form in PHP, render it in Twig, and
let the bundled Stimulus controller handle the JavaScript.** You normally write
no JavaScript at all — the JS assets ship with the package and wire up
automatically (via StimulusBundle / AssetMapper or WebpackEncore).

## Package index

| Package | Install | PHP entry point | Twig render call |
|---|---|---|---|
| Chart.js | `symfony/ux-chartjs` | `ChartBuilderInterface` → `Chart` | `render_chart(chart)` or `<twig:ux:chart>` |
| Cropper.js | `symfony/ux-cropperjs` | `CropperInterface` → `CropperType` form field | `form(form)` (standard form) |
| Dropzone | `symfony/ux-dropzone` | `DropzoneType` form field | `form(form)` (standard form) |
| Notify | `symfony/ux-notify` | `ChatterInterface` + `ChatMessage` + `MercureOptions` | `stream_notifications(['/topic'])` |
| Icons | `symfony/ux-icons` | `ux_icon()` Twig function (PHP: `IconRenderer`) | `ux_icon('name')` or `<twig:ux:icon>` |
| Map | `symfony/ux-map` (+ a renderer bridge) | `Map` object + `Marker`/`InfoWindow`/… | `ux_map(map, {...})` or `<twig:ux:map>` |
| Translator | `symfony/ux-translator` | YAML config; cache warm dumps JS | JS: `import { trans } from './translator'` |
| CalendarLink (experimental) | `symfony/ux-calendar-link` | `CalendarEvent` object | `ux_calendar_link(event, 'google')` / `ux_calendar_links(event)` |

## Rules of thumb

- **Verify the render-call name.** They are not uniform: Chart.js uses
  `render_chart()`, Map uses **`ux_map()`** (not `render_map()`), Icons uses
  `ux_icon()`, Notify uses `stream_notifications()`, CalendarLink uses
  `ux_calendar_link()` / `ux_calendar_links()`. Cropper.js and Dropzone are form
  field types and render through the normal `form()` helper.
- **PHP configures, Twig renders.** Pass options as PHP arrays / objects. For
  Chart.js, the `setData()`/`setOptions()` arrays are handed to Chart.js as-is.
- **HTML attributes go in the render call's second argument** (e.g.
  `render_chart(chart, {'class': '…'})`, `ux_icon('x', {class: '…'})`,
  `ux_map(map, {style: 'height: 300px'})`). To customize behavior, attach a
  custom Stimulus controller via `data-controller` — you don't replace the
  bundled controller, you extend it through its lifecycle events.
- **The HTML/`<twig:ux:*>` component syntax requires `symfony/ux-twig-component`.**
  It exists for Icons, Map and Chart.js. Functionally equivalent to the Twig
  function form.
- **Some bundles need infrastructure:** Notify needs a running **Mercure** hub
  and a configured chatter transport; Map needs a **renderer bridge** package
  (`symfony/ux-google-map` or `symfony/ux-leaflet-map`) selected via a
  `UX_MAP_DSN`. Icons' on-demand fetching needs `symfony/http-client`.
- **Don't hand-edit the generated JS translation files** (Translator) — they are
  produced on cache warm from your Symfony translation catalogs.
- **CalendarLink and Translator are EXPERIMENTAL** — their APIs may change; pin
  versions and re-check on upgrade.
- **Several bundles require StimulusBundle to be configured** before use
  (Cropper.js, Dropzone, Notify, Translator).

## Per-package detail

Longer, copy-ready examples live in the reference files:

- [Chart.js](references/chartjs.md) — building a chart, options, plugins.
- [Cropper.js & Dropzone](references/forms.md) — the two form-field bridges.
- [Notify](references/notify.md) — Mercure-backed browser notifications.
- [Icons](references/icons.md) — `ux_icon()`, Iconify on-demand, local SVGs,
  aliases, accessibility, render precedence.
- [Map](references/map.md) — `Map` object, markers/info windows, renderers,
  shapes, Live Components.
- [Translator & CalendarLink](references/translator-calendarlink.md) — JS
  translations and "Add to calendar" links.
