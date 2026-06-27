---
name: turbo-integration
description: >-
  Integrate Hotwired Turbo into Symfony via the ux-turbo bundle — Turbo Drive
  navigation, Turbo Frames, Turbo Streams from form submits, and Doctrine
  #[Broadcast] live updates over Mercure. Use when building SPA-like
  navigation, partial-page updates, or pushing live model changes to the
  browser without writing JavaScript.
---

# Turbo integration (Symfony UX Turbo)

Symfony UX Turbo (`symfony/ux-turbo`) wraps Hotwired Turbo so you get a
single-page-app feel and live updates with no custom JavaScript. It has three
layers — Drive, Frames, Streams — plus Doctrine broadcasting. Always treat
Turbo as a *progressive enhancement*: the app must still work for clients
without JavaScript.

## Install

```
composer require symfony/ux-turbo
```

With AssetMapper no extra step is needed; with WebpackEncore run the JS build.
Turbo Drive is enabled automatically on install.

## Turbo Drive (page navigation)

Drive intercepts link clicks and form submits, runs them via AJAX, and swaps
the page without a full reload. Three things to get right:

- **Make JS Turbo-ready.** Body `<script>` tags re-execute on every
  navigation. Put scripts in `<head>` with `defer`, and write behavior as
  Stimulus controllers (or equivalent) so it re-binds correctly after swaps.
- **Reload on asset changes.** Enable Encore versioning and add
  `data-turbo-track="reload"` to `<script>`/`<link>` tags (via
  `webpack_encore.script_attributes` / `link_attributes`) so users get fresh
  CSS/JS.
- **Return 422 on invalid form submits.** Drive needs a non-200 to know a
  submit failed and re-render with errors. Symfony's `render()` (6.2+) sets
  this automatically; otherwise build a `Response` with status
  `422` when submitted-and-invalid. Prefer `303` (`HTTP_SEE_OTHER`) for the
  success redirect.
- **Multi-submit forms:** give each `SubmitType` button an explicit `value`
  attribute — Drive does not send buttons with an empty value.

## Turbo Frames (scoped page regions)

Wrap a region in `<turbo-frame id="...">`. A link or form inside the frame
only replaces the matching frame on the target page; the rest of the page is
left alone. The response page must contain a `<turbo-frame>` with the same
`id` — content outside it is ignored.

```html+twig
<turbo-frame id="the_frame_id">
    <a href="{{ path('another-page') }}">Stays scoped to this frame</a>
</turbo-frame>
```

- **Lazy-load** a frame by adding `src="{{ path('block') }}"` (optionally
  `loading="lazy"`); the placeholder content shows until the fragment loads.
- A **Twig component** is available: `<twig:Turbo:Frame id="..." src="..." />`
  renders the `<turbo-frame>` element.
- **Detect frame requests** in a controller by injecting `TurboFrame`
  (`isFrameRequest()`, `getRequestId()`) or in Twig with
  `turbo_is_frame_request()` / `turbo_frame_request_id()`, then render only
  the partial.
- **Keep frame responses minimal.** Frame responses usually skip the full
  layout. To still populate `<head>` (meta tags), extend
  `@Turbo/layouts/frame.html.twig` instead of the app layout.

## Turbo Streams (partial updates)

A Turbo Stream is a `<turbo-stream action="...">` element targeting elements by
CSS selector, with content in a `<template>`. Supported actions: `append`,
`prepend`, `replace`, `update`, `remove`, `before`, `after`, `refresh`.

### Responding to a form submit

Detect the Turbo request format, switch the response format to the stream
format, and render a stream block. Keep the non-Turbo path (redirect) intact:

```php
use Symfony\UX\Turbo\TurboBundle;

if (TurboBundle::STREAM_FORMAT === $request->getPreferredFormat()) {
    $request->setRequestFormat(TurboBundle::STREAM_FORMAT);
    return $this->renderBlock('task/new.html.twig', 'success_stream', ['task' => $task]);
}
return $this->redirectToRoute('task_success', [], Response::HTTP_SEE_OTHER);
```

`TurboBundle::STREAM_FORMAT` corresponds to the
`text/vnd.turbo-stream.html` content type; setting the request format makes
Symfony emit that type so the browser applies the actions instead of doing a
navigation.

### Emitting stream elements

Three equivalent ways to build a `<turbo-stream>`:

- **Raw Twig** in a block, e.g.
  `<turbo-stream action="replace" targets="#id"><template>…</template></turbo-stream>`.
- **Twig components** `<twig:Turbo:Stream:Append target="#id">…`,
  `:Prepend`, `:Replace`, `:Update`, `:Remove`, `:Before`, `:After`,
  `:Refresh`. Add `morph` on `:Replace`/`:Update` to render
  `method="morph"`; `:Refresh requestId="…"` debounces a page refresh.
- **PHP helper** `Symfony\UX\Turbo\Helper\TurboStream` with static methods
  `append`, `prepend`, `replace`, `update`, `remove`, `before`, `after`,
  `refresh`, and a generic `action()`. `replace`/`update` accept a `$morph`
  flag; each returns the `<turbo-stream>` HTML string.

> Only elements in the stream are updated. To reset a form, isolate the form
> rendering into a block and include a fresh (cloned, un-submitted) form in a
> `replace` stream targeting the form.

See [references/streams.md](references/streams.md) for full form/controller
and stream-template examples.

## Doctrine broadcasting with #[Broadcast]

Push entity create/update/remove to all connected clients over Mercure with a
single attribute — no JS.

```php
use Symfony\UX\Turbo\Attribute\Broadcast;

#[ORM\Entity]
#[Broadcast]
class Book { /* … */ }
```

- Requires `symfony/mercure-bundle` and a configured Mercure hub
  (`MERCURE_URL`).
- The entity must expose its identifier: managed by Doctrine ORM, or have a
  public `id` property / `getId()` method.
- **Subscribe** in a template by emitting a stream source. Prefer the Twig
  component `<twig:Turbo:Stream:From topics="App\\Entity\\Book" />` (or
  `turbo_stream_from(...)`), which renders a `<turbo-mercure-stream-source>`.
  `turbo_stream_listen(entity)` / `turbo_stream_listen('App\\Entity\\Book')`
  also works (pass a transport name as the 2nd arg, `withCredentials` for
  private hubs). Manually enabling the `mercure-turbo-stream` controller is
  deprecated since UX 3.1.
- **Template convention:** UX Turbo renders
  `templates/broadcast/{ClassName}.stream.html.twig`, which **must** define
  `create`, `update`, and `remove` blocks (may be empty). Each block holds the
  Turbo Stream actions; actions targeting missing DOM elements are ignored.
  Available variables: `entity`, `id`, `action`, `options`.
- **`#[Broadcast]` options:** `transports` (string[]), `topics` (string[],
  default derived from FQCN + id), `template`. Mercure adds `private`,
  `sse_id`, `sse_type`, `sse_retry`. The attribute `IS_REPEATABLE` — repeat it
  to render different templates/topics for the same change (e.g. list vs.
  detail). Mercure topics support Expression Language when prefixed with `@=`.

See [references/broadcast.md](references/broadcast.md) for the broadcast
template and a Mercure chat example.

## Testing & interactivity caveats

- Stream/Frame behavior runs in **JavaScript**, so it is not exercised by
  BrowserKit/`WebTestCase`. Use a real browser via **Symfony Panther**
  (`PantherTestCase`, `assertSelectorWillContain`, etc.).
- Because navigation no longer reloads the page, **any custom interactivity
  must survive Turbo swaps** — write it as Stimulus controllers rather than
  one-off inline scripts, and keep `<script>` out of `<body>`.

## Rules of thumb

- Keep the no-JS fallback working: a stream branch and a redirect branch in the
  same action.
- Pick the smallest tool: Drive for whole-page nav, Frames for an isolated
  region, Streams for targeted multi-element updates, `#[Broadcast]` for
  live pushes to all clients.
- Match `<turbo-frame>` / `<turbo-stream>` targets to real `id`/selectors; a
  stream against a missing element is silently dropped.
