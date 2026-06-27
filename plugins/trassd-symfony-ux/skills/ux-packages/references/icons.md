# Icons (`symfony/ux-icons`)

Renders SVG icons — both local files and remote icons from popular sets (via
Iconify) — through a Twig function. No JavaScript involved; the SVG is embedded
into the HTML at render time by the PHP `IconRenderer`.

## Rendering

```twig
{# embeds assets/icons/user-profile.svg #}
{{ ux_icon('user-profile') }}

{# subdirectory icon assets/icons/admin/user-profile.svg #}
{{ ux_icon('admin:user-profile') }}

{# downloads + embeds an icon from a remote set via ux.symfony.com (Iconify) #}
{{ ux_icon('flowbite:user-solid') }}

{# second argument = HTML attributes on the <svg> #}
{{ ux_icon('user-profile', {class: 'w-4 h-4'}) }}
```

HTML/component syntax (needs `symfony/ux-twig-component`):

```html+twig
<twig:ux:icon name="user-profile" class="w-4 h-4" />
```

Note: the `<twig:ux:icon>` component does **not** support embedded content —
any children between the tags are ignored.

## Icon names

- `prefix:name` (e.g. `mdi:check`, `bi:check`) or just `name`.
- The icon `name` is the file name without extension (`check.svg` → `check`),
  and must be a slug (`[a-z0-9-]+`).
- Depending on configuration, the `prefix` is an icon-set identifier and/or a
  subdirectory under the icon directory.

## Where icons come from

1. **Local SVGs** — drop files in `assets/icons/` (subdirectories become the
   prefix) and commit them.
2. **On-demand** — paste a snippet from `ux.symfony.com/icons`; the icon is
   fetched from the Iconify API and cached. Requires `symfony/http-client`.
3. **Imported (locked)** — `php bin/console ux:icons:import flowbite:user-solid`
   downloads the icon into `assets/icons/` so it is version-pinned and committed
   (like a lockfile). `ux:icons:lock` imports all on-demand icons currently used.

**Local icons of the same name take precedence over on-demand icons.**

## Sizing & color

- `<svg>` is browser-sized; set a size explicitly. Use `em` units to track text
  (`{style: 'height: 1em;'}`), or your CSS framework's utilities (`{class: 'size-4'}`).
- Icons render with `fill="currentColor"`, so they inherit the CSS `color` of
  the surrounding element; override per-icon with a class.

## Configuration highlights (`config/packages/ux_icons.yaml`)

- `default_icon_attributes` — attributes added to every icon.
- `aliases` — shortcut names mapping to real icon ids (e.g. `dots: 'clarity:ellipsis-horizontal-line'`).
- `icon_sets` with per-set `icon_attributes` and `suffixes` (suffix-based
  attribute overrides; longest suffix wins, `''` is the fallback).
- `ignore_not_found: true` — render nothing instead of throwing in production.

**Attribute precedence (lowest → highest):** the icon file's own attributes <
renderer configuration (defaults, set, suffix attributes) < attributes passed in
the `ux_icon()` call.

## Accessibility

The renderer automatically adds `aria-hidden="true"` to any icon that has none
of `aria-label`, `aria-labelledby`, or `title`. Provide `aria-label` for
informative/functional icons; set `aria-hidden="false"` to opt out for a
specific icon.

## Caching

Icons are cached. Pre-warm in production with `php bin/console ux:icons:warm-cache`
(automatic during `asset-map:compile` when using AssetMapper). Cache warming
scans templates for `something:something` strings, so **use string-literal icon
names** — dynamically built names (`'flag-' ~ locale`) are not cached.
