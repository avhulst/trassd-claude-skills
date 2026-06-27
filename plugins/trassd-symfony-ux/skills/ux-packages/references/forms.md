# Form-field bridges: Cropper.js & Dropzone

Both expose a Symfony **form type** and render through the normal `{{ form(form) }}`
helper — no special Twig render function. Both require StimulusBundle to be
configured first.

## Cropper.js (`symfony/ux-cropperjs`)

Inject `CropperInterface`, create a `Crop` from a server-side image path,
configure it, and add it to a form with `CropperType`.

```php
use Symfony\UX\Cropperjs\Factory\CropperInterface;
use Symfony\UX\Cropperjs\Form\CropperType;

public function index(CropperInterface $cropper, Request $request): Response
{
    $crop = $cropper->createCrop('/server/path/to/the/image.jpg');
    $crop->setCroppedMaxSize(2000, 1500);

    $form = $this->createFormBuilder(['crop' => $crop])
        ->add('crop', CropperType::class, [
            'public_url' => '/public/url/to/the/image.jpg',
            'cropper_options' => ['aspectRatio' => 2000 / 1500],
        ])
        ->getForm();

    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
        $crop->getCroppedImage();              // cropped image data (string)
        $crop->getCroppedThumbnail(200, 150);  // thumbnail (string)
    }

    return $this->render('home/index.html.twig', ['form' => $form->createView()]);
}
```

- `public_url` is the browser-reachable URL of the image to crop.
- `cropper_options` accepts any Cropper.js option (e.g. `aspectRatio`).
- The result is read back from the `Crop` object after the form is submitted.

Render it like any form: `{{ form(form) }}`.

To extend behavior, pass `'attr' => ['data-controller' => 'mycropper']` to the
field and listen for `cropperjs:pre-connect` / `cropperjs:connect` in your
Stimulus controller (`event.detail.cropper`, `.options`, `.img`).

## Dropzone (`symfony/ux-dropzone`)

A drag-and-drop replacement for the native `FileType`. Use `DropzoneType` in a
form type:

```php
use Symfony\UX\Dropzone\Form\DropzoneType;

public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder->add('photo', DropzoneType::class);
}
```

- **Styling:** a default stylesheet is auto-imported. To use your own CSS, set
  the `@symfony/ux-dropzone/dist/style.min.css` autoimport to `false` in
  `assets/controllers.json` (set to `false`, don't delete the line, so Flex
  won't re-add it).
- **Extending:** pass `'attr' => ['data-controller' => 'mydropzone']` and listen
  for `dropzone:connect`, `dropzone:change`, `dropzone:clear`.
