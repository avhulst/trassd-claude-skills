# Image formats — meta titles, compression, transformations

All examples live in `config/image-formats.xml` under the root
`<formats>` element:

```xml
<formats xmlns="http://schemas.sulu.io/media/formats"
         xsi:schemaLocation="http://schemas.sulu.io/media/formats http://schemas.sulu.io/media/formats-1.1.xsd">
    <!-- formats go here -->
</formats>
```

## Meta titles

Provide a localized title per format — shown to content managers in the admin
when they crop images:

```xml
<format key="300x">
    <meta>
        <title lang="en">Author Avatar</title>
        <title lang="de">Autor Avatar</title>
    </meta>
    <scale x="300"/>
</format>
```

## Per-format compression

Override the global compression on a single format with `<options>`:

```xml
<format key="300x">
    <scale x="300"/>
    <options>
        <option name="jpeg_quality">80</option>
        <option name="webp_quality">80</option>
        <option name="avif_quality">80</option>
        <option name="png_compression_level">6</option>
    </options>
</format>
```

## Transformations

Transformations add effects on top of the scaled image. They live in a
`<transformations>` block; each `<transformation>` has an `<effect>` and
optional `<parameters>`.

### Blur — blurs by a `sigma` parameter

```xml
<format key="300x-blur">
    <scale x="300"/>
    <transformations>
        <transformation>
            <effect>blur</effect>
            <parameters>
                <parameter name="sigma">6</parameter>
            </parameters>
        </transformation>
    </transformations>
</format>
```

### Grayscale — converts to black/white

```xml
<format key="300x-black">
    <scale x="300"/>
    <transformations>
        <transformation>
            <effect>grayscale</effect>
        </transformation>
    </transformations>
</format>
```

### Gamma — applies a gamma `correction`

```xml
<format key="300x-gamma">
    <scale x="300"/>
    <transformations>
        <transformation>
            <effect>gamma</effect>
            <parameters>
                <parameter name="correction">0.7</parameter>
            </parameters>
        </transformation>
    </transformations>
</format>
```

### Sharpen — applies a sharpen effect

```xml
<format key="300x-sharpen">
    <scale x="300"/>
    <transformations>
        <transformation>
            <effect>sharpen</effect>
        </transformation>
    </transformations>
</format>
```

### Paste — overlays another image (e.g. border or copyright)

Reference the overlay with the `image` parameter; position it with optional
`x`, `y`, `w`, `h` parameters:

```xml
<format key="300x300-border">
    <scale x="300" y="300"/>
    <transformations>
        <transformation>
            <effect>paste</effect>
            <parameters>
                <parameter name="image">@AppBundle/Resources/public/border.png</parameter>
                <parameter name="x">0</parameter>
                <parameter name="y">0</parameter>
                <parameter name="w">300</parameter>
                <parameter name="h">300</parameter>
            </parameters>
        </transformation>
    </transformations>
</format>
```

### Combining transformations

List multiple `<transformation>` entries; they apply in order:

```xml
<format key="300x-blur-black">
    <scale x="300"/>
    <transformations>
        <transformation>
            <effect>blur</effect>
            <parameters>
                <parameter name="sigma">6</parameter>
            </parameters>
        </transformation>
        <transformation>
            <effect>grayscale</effect>
        </transformation>
    </transformations>
</format>
```

## Regenerating after changes

When you edit existing formats, regenerate already-generated images:

```bash
php bin/websiteconsole sulu:media:regenerate-formats
```

Or clear the format cache so images regenerate lazily on next request:

```bash
php bin/websiteconsole sulu:media:format:cache:clear
```

Remove generated formats whose media no longer exist in the database (e.g.
multi-server cleanup):

```bash
php bin/websiteconsole sulu:media:format:cache:cleanup
```
