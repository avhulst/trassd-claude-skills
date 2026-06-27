# Turbo Streams — form responses & stream templates

## Controller: stream on submit, redirect otherwise

```php
namespace App\Controller;

use App\Entity\Task;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\UX\Turbo\TurboBundle;

class TaskController extends AbstractController
{
    public function new(Request $request): Response
    {
        $form = $this->createForm(TaskType::class, new Task());
        $emptyForm = clone $form;              // for resetting the form (see below)
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $task = $form->getData();
            // ... persist ...

            if (TurboBundle::STREAM_FORMAT === $request->getPreferredFormat()) {
                $request->setRequestFormat(TurboBundle::STREAM_FORMAT);
                return $this->renderBlock('task/new.html.twig', 'success_stream', [
                    'task' => $task,
                    'form' => $emptyForm,
                ]);
            }

            // No-JS / non-Turbo fallback
            return $this->redirectToRoute('task_success', [], Response::HTTP_SEE_OTHER);
        }

        return $this->render('task/new.html.twig', ['form' => $form]);
    }
}
```

`TurboBundle::STREAM_FORMAT` maps to the `text/vnd.turbo-stream.html`
content type. Setting the request format makes Symfony emit that type so the
browser applies the stream actions instead of navigating.

## Stream block in the template

```html+twig
{# new.html.twig #}
{% block task_form %}
    {{ form(form) }}
{% endblock %}

{% block success_stream %}
    {# replace the form with a fresh one so it resets #}
    <turbo-stream action="replace" targets="form[name=task]">
        <template>{{ block('task_form') }}</template>
    </turbo-stream>

    {# update some other region of the page #}
    <turbo-stream action="replace" targets="#my_div_id">
        <template>
            <div>The task "{{ task }}" has been created!</div>
        </template>
    </turbo-stream>
{% endblock %}
```

Only elements named in the stream are updated. To reset a form, clone the
un-submitted form in the controller, pass it into the stream, and `replace`
the rendered form block.

## Twig stream components

Equivalent to writing the raw element; the component injects content into the
`<template>`:

```html+twig
<twig:Turbo:Stream:Append  target="#dom_id">…</twig:Turbo:Stream:Append>
<twig:Turbo:Stream:Prepend target="#dom_id">…</twig:Turbo:Stream:Prepend>
<twig:Turbo:Stream:Replace target="#dom_id">…</twig:Turbo:Stream:Replace>
<twig:Turbo:Stream:Update  target="#dom_id">…</twig:Turbo:Stream:Update>
<twig:Turbo:Stream:Before  target="#dom_id">…</twig:Turbo:Stream:Before>
<twig:Turbo:Stream:After   target="#dom_id">…</twig:Turbo:Stream:After>
<twig:Turbo:Stream:Remove  target="#dom_id" />
<twig:Turbo:Stream:Refresh requestId="abcd-1234" />

{# morphing: emits method="morph" #}
<twig:Turbo:Stream:Replace target="#dom_id" morph>…</twig:Turbo:Stream:Replace>
```

## PHP helper

`Symfony\UX\Turbo\Helper\TurboStream` returns the `<turbo-stream>` HTML string:

```php
use Symfony\UX\Turbo\Helper\TurboStream;

TurboStream::append('#messages', $html);
TurboStream::prepend('#messages', $html);
TurboStream::replace('#item_1', $html, morph: true);
TurboStream::update('#item_1', $html);
TurboStream::remove('#item_1');
TurboStream::before('#item_1', $html);
TurboStream::after('#item_1', $html);
TurboStream::refresh();                 // or refresh('abcd-1234') to debounce
TurboStream::action('custom', '#id', $html, ['data-x' => '1']);
```
