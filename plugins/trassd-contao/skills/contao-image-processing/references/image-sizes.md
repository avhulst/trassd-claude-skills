# Image sizes — reference

Predefine image sizes in `config/config.yaml` under `contao.image.sizes` (no
database needed) so they show up in the back end and can be referenced by an
underscore-prefixed key. Sizes can also be created in the back end as
`tl_image_size` records and referenced by integer ID.

Run `vendor/bin/contao-console config:dump-reference contao` to see the full
image-size option reference.

## Basic configuration

```yaml
contao:
    image:
        sizes:
            example:
                width: 512
            foobar:
                width: 1024
```

A richer example — crop to 128×128, zoom into the important part, add `1.5x`/`2x`
densities and an `<img>` CSS class:

```yaml
contao:
    image:
        sizes:
            example:
                width: 128
                height: 128
                resize_mode: crop
                zoom: 100
                css_class: example
                lazy_loading: true
                densities: 1.5x, 2x
```

Notes:

- The `1x` density (original size) is always generated even if not listed.
- By default an image is served from `assets/images/` even when no resize is
  needed; set `skip_if_dimensions_match` to change this.

## Media queries (multiple `<source>`)

Use `items` to add media-query-specific sources:

```yaml
contao:
    image:
        sizes:
            example:
                width: 1024
                height: 512
                resize_mode: box
                densities: 1.25x
                css_class: example
                items:
                    -
                        media: '(max-width: 512px)'
                        width: 128
                        height: 64
                        resize_mode: box
                        densities: 2x
                    -
                        media: '(max-width: 1024px)'
                        width: 512
                        height: 256
                        resize_mode: box
                        densities: 1.5x
```

## Format conversion

Per source format, define target formats; a `<picture>` is generated with all
targets, the last being the `<img>` fallback:

```yaml
contao:
    image:
        sizes:
            example:
                width: 256
                height: 256
                resize_mode: crop
                formats:
                    jpg: [webp, jpg]
                    webp: [webp, jpg]
                    png: [webp, png]
```

## Defaults

Share common settings across sizes with the `_defaults` key; individual sizes
omit or override them:

```yaml
contao:
    image:
        sizes:
            _defaults:
                formats:
                    jpg: [webp, jpg]
                densities: 0.75x, 2x
                lazy_loading: true
                resize_mode: crop
            large_photo:
                width: 1000
                height: 500
            small_box:
                densities: 2x
                resize_mode: box
                width: 100
                height: 100
```

## Translating labels

Sizes appear in the back end under their config key. Provide friendlier labels
via the `image_sizes` translation domain:

```yaml
# translations/image_sizes.en.yaml
example: Image with 512 Pixel width
foobar: Image with 1024 Pixel width
```

## Size array formats

Accepted by the factories, the Image Studio's `setSize`, and the legacy
`Controller::addImageToTemplate`:

```php
// [width, height, mode] — mode is a ResizeConfiguration::MODE_* constant or string
$size = [256, 128, 'crop'];
$size = [256, 128, ResizeConfiguration::MODE_BOX];

// [0, 0, <tl_image_size ID>] — fetch a DB-stored configuration; width/height ignored
$size = [0, 0, 8];

// [0, 0, '_configKey'] — reference a contao.image.sizes key; underscore prefix required
$size = [0, 0, '_example'];
```

Modes: `crop`, `box`, `proportional`.
