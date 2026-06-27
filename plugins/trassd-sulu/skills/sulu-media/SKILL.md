---
name: sulu-media
description: >-
  Handle media in Sulu — image formats, media configuration, external storage
  adapters, and properties providers. Triggers when configuring image formats
  (config/image-formats.xml), tuning upload limits or the imagine adapter,
  integrating external/cloud media storage via Flysystem, rendering thumbnails
  in Twig, or registering a custom media properties provider in the MediaBundle.
---

# Sulu Media

The **SuluMediaBundle** handles all media assets: upload, storage, nestable
collections, and automatic image resizing/scaling. Media are exposed in Twig and
PHP through their `FileVersion` data (title, thumbnails, and provider-supplied
properties such as width/height).

## Image formats

Image formats let you output uploaded images at the exact dimensions a template
needs, and let content managers pick the cutout per format. Formats are
generated **on demand** — only when first requested — to save disk space.

Define formats in **`config/image-formats.xml`** (root `<formats>`, namespace
`http://schemas.sulu.io/media/formats`, schema `formats-1.1.xsd`). Each format
has a unique `key` and a `<scale>`:

```xml
<formats xmlns="http://schemas.sulu.io/media/formats"
         xsi:schemaLocation="http://schemas.sulu.io/media/formats http://schemas.sulu.io/media/formats-1.1.xsd">
    <format key="300x"><scale x="300"/></format>          <!-- fixed width, dynamic height -->
    <format key="x200"><scale y="200"/></format>          <!-- dynamic width, fixed height -->
    <format key="300x200"><scale x="300" y="200"/></format>
    <format key="300x200-inset"><scale x="300" y="200" mode="inset"/></format> <!-- max w/max h -->
</formats>
```

Rules and tips:

- `mode="inset"` treats `x`/`y` as **maximums** (fit within), not a fixed crop.
- Sulu **never upscales**: if the original is smaller than the requested format,
  it returns the original in the format's aspect ratio. Scale up to the HTML
  container on the frontend with CSS (e.g. `width: 100%;`).
- **Define a `<meta><title lang="...">`** for every format — these titles are
  shown to content managers in the admin cropping UI.
- After **editing an existing format**, regenerate already-generated images:
  `php bin/websiteconsole sulu:media:regenerate-formats`, or clear the format
  cache (`sulu:media:format:cache:clear`) to regenerate lazily on next request.
- Remove orphaned formats (media deleted in the admin, e.g. multi-server):
  `php bin/websiteconsole sulu:media:format:cache:cleanup`.

For meta titles, per-format compression `<options>`, and all transformation
effects (`blur`, `grayscale`, `gamma`, `sharpen`, `paste`, and combining them),
see [references/image-formats.md](references/image-formats.md).

### Rendering formats in Twig

Format URLs reach the template via the media's **`thumbnails`** property, keyed
by format key:

```twig
<img src="{{ image.thumbnails['200x100'] }}" alt="{{ image.title }}"/>
```

Output defaults to the original file's format. Request a specific file format by
appending its extension to the format key (e.g. `png`, `webp`):

```twig
<img src="{{ image.thumbnails['200x100.webp'] }}" alt="{{ image.title }}"/>
```

### Image compression

Images are **not** compressed on upload by default. Set compression globally in
`config/packages/sulu_media.yaml` (create the file if absent) — keep
`jpeg_quality` around 70–90:

```yaml
sulu_media:
    format_manager:
        default_imagine_options:
            jpeg_quality: 80
            webp_quality: 80
            avif_quality: 80
            png_compression_level: 6
```

Per-format compression goes in that format's `<options>` (see the reference).

## MediaBundle configuration

Configure in `config/packages/sulu_media.yaml`:

```yaml
sulu_media:
    adapter: 'auto'   # imagine adapter; or fix to 'gd', 'vips', or 'imagick'
    upload:
        max_filesize: 256          # max upload size in MB
        blocked_file_types:        # mime types users may not upload
            - video/x-flv
            - video/mp4
            - video/quicktime
```

The `adapter` selects the imagine adapter used to process images; properties
providers (below) also depend on it.

## External / cloud storage

Sulu stores media through the **Flysystem** filesystem abstraction
(`flysystem-bundle`), so backends are swappable. The default is local storage.
Configure a storage in `config/packages/flysystem.yaml`, then point Sulu at it
via `sulu_media.storage.flysystem_service`:

```yaml
# config/packages/flysystem.yaml
flysystem:
    storages:
        default.storage:
            adapter: 'local'
            options:
                directory: '%kernel.project_dir%/var/storage/default'

# config/packages/sulu_media.yaml
sulu_media:
    storage:
        flysystem_service: 'default.storage'
```

Swap `adapter` (and add the matching Flysystem package) for S3, Google Cloud
Storage, Azure Blob, etc. — see the flysystem-bundle cloud-storage docs.
Internally Sulu's `FlysystemStorage` implements its `StorageInterface`.

### Image formats with external storage

**Critical caveat:** only **original files** go to the external storage. Image
formats / thumbnails are generated **on demand into the local public directory**
(`/uploads/media/*`), where the web server serves them as a proxy on repeat
requests. S3 / GCS / Azure provide no such on-demand proxy.

- To serve formats from the cloud, front `/uploads/media/*` with a CDN/proxy
  (Fastly, Cloudflare, …) that caches generated formats long-term. For a custom
  CDN domain, wrap URLs with Symfony's asset helper —
  `{{ asset(media.thumbnail['40x40']) }}` — and configure `framework.assets`.
- Only after verifying the CDN caches correctly may you stop writing thumbnails
  locally, via `sulu_media.format_cache.save_image: false` (drive it with an ENV
  var so local dev still saves them).
- **Never disable the format cache without a CDN/proxy** — the server would
  regenerate every format on every request, which is resource-intensive and can
  overwhelm it.

## Media properties providers

Properties providers gather data for an uploaded file; Sulu saves the result on
the `FileVersion`, accessible in Twig and PHP. Built-in providers:

- **`ImagePropertiesProvider`** — for any image the imagine adapter supports;
  provides `width` and `height` (useful for CSS placeholders / layout).
- **`VideoPropertiesProvider`** — requires `ffprobe` installed/configured;
  provides `duration`, `width`, and `height`.

### Custom provider

Implement **`MediaPropertiesProviderInterface`** (method `provide(File $file): array`)
and return an array of properties:

```php
use Sulu\Bundle\MediaBundle\Media\PropertiesProvider\MediaPropertiesProviderInterface;
use Symfony\Component\HttpFoundation\File\File;

class ExifPropertiesProvider implements MediaPropertiesProviderInterface
{
    public function provide(File $file): array
    {
        if (!\fnmatch('image/*', (string) $file->getMimeType())) {
            return [];
        }
        // read and return properties, e.g. ['exif_header_name' => ...]
        return [];
    }
}
```

With autoconfiguration the service is picked up automatically. **When
`autoconfigure` is disabled**, tag it `sulu_media.media_properties_provider`:

```yaml
App\Media\PropertiesProvider\ExifPropertiesProvider:
    tags:
        - { name: 'sulu_media.media_properties_provider' }
```

Verify registration:
`php bin/adminconsole debug:container --tag sulu_media.media_properties_provider`.
