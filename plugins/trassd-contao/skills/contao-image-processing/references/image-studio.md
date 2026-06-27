# Image Studio — FigureBuilder reference

The `Contao\CoreBundle\Image\Studio\Studio` service (`contao.image.studio`) is a
factory that produces:

- **`FigureBuilder`** — fluent builder for `Figure` objects (`createFigureBuilder()`).
- **`ImageResult`** — picture/image data (source/img/original dimensions),
  lazily querying the image and picture factories. Always part of a `Figure`.
- **`LightboxResult`** — lightbox data (group identifier, link, optional resized
  lightbox `ImageResult`). Can be part of a `Figure`.

## Building a Figure

```php
$figureBuilder = $this->studio->createFigureBuilder();

// From legacy code:
$figureBuilder = System::getContainer()
    ->get(Contao\CoreBundle\Image\Studio\Studio::class)
    ->createFigureBuilder();

$figure = $figureBuilder
    ->fromPath('/path/to/my/file.png')
    ->setSize('_my_size')
    ->setMetadata($metadata)
    ->setLinkHref('https://example.com')
    ->setLinkAttribute('data-foo', 'bar')
    ->setOptions(['attr' => ['class' => 'custom-figure']])
    ->build();              // throws if resource missing
    // ->buildIfResourceExists();  // returns null instead
```

After building you may reconfigure (repeat the setters) and build again — useful
for galleries that share most of their configuration.

## Defining the base resource

| Method | Purpose |
|-|-|
| `fromFilesModel` | From a `FilesModel`. |
| `fromUuid` | From a `tl_files` UUID. |
| `fromId` | From a `tl_files` ID. |
| `fromPath` | From an absolute/relative path. `$autoDetectDbafsPaths = false` skips the `FilesModel` lookup. |
| `fromImage` | From an `ImageInterface` (tries to find a matching `FilesModel`). |
| `from` | Auto-detect the type and resolve to one of the above. |

## Setting options

| Method | Purpose |
|-|-|
| `setSize` | A `PictureConfiguration`, a size array, a `tl_image_size` ID, or a `_configKey`. `null` disables resizing. |
| `setMetadata` | Overwrite the metadata derived from the resource's `FilesModel`. `null` restores default. |
| `disableMetadata` | Skip metadata entirely. `false` restores default. |
| `setLocale` | Locale for metadata. Required when used outside a request context and metadata is needed. |
| `setLinkAttribute` / `setLinkAttributes` | Add/merge custom link attributes (override auto-generated ones; `null` removes). |
| `setLinkHref` | Shorthand for the `href` link attribute. |
| `enableLightbox` | Enable lightbox creation (off by default; `false` restores default). |
| `setLightboxResourceOrUrl` | Override the derived lightbox resource (path or `ImageInterface`). |
| `setLightboxSize` | Override the lightbox size. Required outside a request context. |
| `setLightboxGroupIdentifier` | Set `data-lightbox="<group>"`. |
| `setOptions` | Associative array passed verbatim to the template, e.g. `['attr' => ['class' => 'my_figure']]`. |

## Consuming a Figure

A `Figure` groups everything about an image:

- `getImage(): ImageResult` — the resized image/picture data for markup.
- `hasMetadata(): bool` / `getMetadata(): ?Metadata` — alt text, caption, etc.
- `hasLightbox(): bool` / `getLightbox(): ?LightboxResult`.
- `getLinkAttributes(bool $includeHref = false)` / `getLinkHref(): string`.

## Templating

In Twig, render with the `figure()` function or the `figure` / `picture`
components, which output a `<figure>` with the image and its metadata. This is
the most versatile and recommended approach in Contao 5; see the Twig template
documentation's image section for full examples.
