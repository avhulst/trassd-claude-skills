---
name: contao-image-processing
description: Process images in Contao — the image/picture factories, the Image Studio, and predefined image sizes. Triggers when resizing/rendering images, using the Image Studio, or defining image sizes in a Contao extension.
---

# Contao Image Processing

Contao processes and resizes images through a layered stack built on the
`contao/image` library (which wraps `imagine/imagine` and `contao/imagine-svg`).
Choose the highest abstraction that fits the job:

| Use case | Component | Service |
|-|-|-|
| Render an image in a template (preferred) | Image Studio `FigureBuilder` | `contao.image.studio` |
| Responsive `<picture>` with full control | `PictureFactory` | `contao.image.picture_factory` |
| Single resized image with full control | `ImageFactory` | `contao.image.factory` |

## Rules

- **Prefer the Image Studio** for anything that ends up in a template. It wires
  up metadata, lightbox and link handling that the raw factories do not.
- **Inject the interface, not the concrete class**, for the factories:
  `Contao\CoreBundle\Image\ImageFactoryInterface` and
  `Contao\CoreBundle\Image\PictureFactoryInterface`. For the Studio, inject
  `Contao\CoreBundle\Image\Studio\Studio` (autowiring resolves the service).
- **Never build resized files by hand** with GD/Imagine directly — go through
  the factories or Studio so caching, deferred resizing and `assets/images/`
  output are handled for you.
- **Define reusable sizes in configuration** (`contao.image.sizes`) instead of
  repeating size arrays. Reference them by `_key` everywhere.
- **Resize modes** are the `ResizeConfiguration::MODE_*` constants:
  `crop`, `box`, `proportional`. Use the string or the constant interchangeably
  in a size array.

## Image Studio (preferred)

`contao.image.studio` (`Contao\CoreBundle\Image\Studio\Studio`) is a factory.
Call `createFigureBuilder()` to get a fluent `FigureBuilder`, configure it, then
`build()` (or `buildIfResourceExists()` to get `null` instead of an exception
when the resource is missing). The result is a `Figure`.

```php
use Contao\CoreBundle\Image\Studio\Studio;

public function __construct(private readonly Studio $studio) {}

public function render(string $uuid): void
{
    $figure = $this->studio
        ->createFigureBuilder()
        ->fromUuid($uuid)
        ->setSize([800, 600, 'crop'])
        ->enableLightbox()
        ->buildIfResourceExists();

    if (null === $figure) {
        return; // resource does not exist
    }
}
```

- **Pick a base resource** with one `from*` method: `fromUuid`, `fromId`,
  `fromPath`, `fromFilesModel`, `fromImage`, or `from` (auto-detects).
- **`setSize`** accepts a `PictureConfiguration`, a [size array](#image-sizes),
  a `tl_image_size` ID, or an underscore-prefixed config key (`_my_size`).
  Pass `null` to disable resizing.
- A `Figure` exposes `getImage()` (an `ImageResult` for `<img>`/`<picture>`
  markup), `hasMetadata()` / `getMetadata()`, `hasLightbox()` / `getLightbox()`,
  and `getLinkAttributes()` / `getLinkHref()`.
- Render a `Figure` in Twig with the `figure()` function / `figure` component.
  After building you can reconfigure and rebuild for galleries.

See [references/image-studio.md](references/image-studio.md) for the full
`FigureBuilder` option reference (metadata, lightbox, links, options).

## Factories (full control)

Use the factories when you need a raw `ImageInterface` or `PictureInterface`
outside the templating flow.

```php
use Contao\CoreBundle\Image\ImageFactoryInterface;

public function __construct(private readonly ImageFactoryInterface $imageFactory) {}

$image = $this->imageFactory->create(
    '/path/to/image.jpg',
    [100, 100, ResizeConfiguration::MODE_CROP],
);

echo $image->getUrl('/root');                       // assets/images/9/image-….jpg
echo $image->getDimensions()->getSize()->getWidth(); // 100
```

- `ImageFactory::create($path, $size, $options)` returns an `ImageInterface`
  (possibly a `DeferredImageInterface` if no target path is given and the file
  does not yet exist). The third argument is a target-path string or a
  `ResizeOptions`.
- `PictureFactory::create($path, $size)` returns a `PictureInterface` for
  responsive `<picture>`/`srcset`/`sizes` output.
- Each argument accepts shorthand or objects — see
  [references/factories.md](references/factories.md) for the equivalent
  `ResizeConfiguration` / `PictureConfiguration` long forms.

## Image sizes

Predefine sizes in `config/config.yaml` under `contao.image.sizes` so they
appear in the back end and can be referenced by key:

```yaml
contao:
    image:
        sizes:
            example:
                width: 128
                height: 128
                resize_mode: crop      # crop | box | proportional
                zoom: 100
                css_class: example
                lazy_loading: true
                densities: 1.5x, 2x
```

- The `1x` density is always generated. Use `skip_if_dimensions_match` to avoid
  serving from `assets/images/` when no resizing is needed.
- `items` adds media-query `<source>` variants; `formats` enables automatic
  format conversion (e.g. `jpg: [webp, jpg]`). `_defaults` shares settings
  across sizes. Translate back-end labels via the `image_sizes` domain.
- A **size array** has these forms: `[width, height, mode]`,
  `[0, 0, <tl_image_size ID>]`, or `[0, 0, '_configKey']` (underscore prefix
  distinguishes config keys from resize modes).

See [references/image-sizes.md](references/image-sizes.md) for media-query,
format-conversion, defaults, and translation examples.
