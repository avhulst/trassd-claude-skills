# Image & Picture factories — reference

Use these when you need a raw `Contao\Image\ImageInterface` or
`Contao\Image\PictureInterface` outside the templating flow. For templates,
prefer the Image Studio.

## ImageFactory

Service `contao.image.factory` (`contao.image.image_factory` before Contao 5),
implementing `Contao\CoreBundle\Image\ImageFactoryInterface`. Retrieves a single
resized image.

`create($path, $size, $options)` accepts shorthand:

- `$path`: a `string` path or an `ImageInterface`.
- `$size`: a size array, a `tl_image_size` integer ID, a `_configKey` string, or
  a `ResizeConfiguration`.
- `$options`: a target-path `string` or a `ResizeOptions`.

```php
use Contao\CoreBundle\Image\ImageFactoryInterface;

public function __construct(private readonly ImageFactoryInterface $imageFactory) {}

$image = $this->imageFactory->create(
    '/path/to/image.jpg',
    [100, 100, ResizeConfiguration::MODE_CROP],
);
```

The shorthand above is equivalent to the explicit object form:

```php
$image = $this->imageFactory->create(
    new Image('/path/to/image.jpg', $imagineService),
    (new ResizeConfiguration())
        ->setWidth(100)
        ->setHeight(100)
        ->setMode(ResizeConfiguration::MODE_CROP),
);
```

Reading the result:

```php
echo $image->getPath();                              // /root/assets/images/9/image-….jpg
echo $image->getUrl('/root');                        // assets/images/9/image-….jpg
echo $image->getDimensions()->getSize()->getWidth(); // 100
```

Specify a target path either as the third string argument or via
`ResizeOptions`:

```php
$image = $this->imageFactory->create(
    '/path/to/source/image.jpg',
    [100, 100, ResizeConfiguration::MODE_CROP],
    '/path/to/target/image.jpg',
);

$image = $this->imageFactory->create(
    '/path/to/image.jpg',
    [100, 100, ResizeConfiguration::MODE_CROP],
    (new ResizeOptions())->setTargetPath('/path/to/target/image.jpg'),
);
```

Without a target path, if the resized file does not yet exist you may receive a
`Contao\Image\DeferredImageInterface` — same metadata, but the file is generated
later.

## PictureFactory

Service `contao.image.picture_factory`, implementing
`Contao\CoreBundle\Image\PictureFactoryInterface`. Produces a responsive image
(`PictureInterface`) meant for `<picture>`, `srcset` and `sizes`.

`create($path, $size)` accepts the same shorthand; `$size` may additionally be a
`PictureConfiguration`.

```php
use Contao\CoreBundle\Image\PictureFactoryInterface;

public function __construct(private readonly PictureFactoryInterface $pictureFactory) {}

$picture = $this->pictureFactory->create(
    '/path/to/image.jpg',
    [100, 100, ResizeConfiguration::MODE_CROP],
);
```

Equivalent explicit form:

```php
$picture = $this->pictureFactory->create(
    new Image('/path/to/image.jpg', $imagineService),
    (new PictureConfiguration())
        ->setSize((new PictureConfigurationItem())
            ->setResizeConfig((new ResizeConfiguration())
                ->setWidth(100)
                ->setHeight(100)
                ->setMode(ResizeConfiguration::MODE_CROP)
            )
        )
);
```
