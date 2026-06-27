---
name: stimulus-controllers
description: >-
  Best practices for writing and wiring Stimulus controllers in a Symfony app via StimulusBundle —
  controllers, targets, values, CSS classes, outlets, the stimulus_controller()/stimulus_action()/stimulus_target()
  Twig helpers, the assets/controllers.json + assets/controllers/ layout, and asset loading through AssetMapper or
  Webpack Encore. Triggers when adding JavaScript behavior to a Symfony app, creating or editing a controller under
  assets/controllers/, registering a third-party UX controller in assets/controllers.json, or using data-controller /
  data-action / data-*-target or the stimulus_* Twig helpers in templates.
---

# Stimulus Controllers (StimulusBundle)

StimulusBundle integrates [Stimulus](https://stimulus.hotwired.dev/) with Symfony: it
auto-registers your custom controllers, activates third-party UX-package controllers, and
ships Twig helpers for emitting the `data-*` attributes Stimulus reads.

## Setup essentials

- Install with `composer require symfony/stimulus-bundle`. With Symfony Flex the recipe wires
  everything up; otherwise follow the manual setup.
- You need an asset system first — either **AssetMapper** (PHP-based, no Node) or **Webpack
  Encore** (Node-based). Both work; choose one per project.
- The recipe scaffolds:
  - `assets/controllers/` — your custom controllers live here (ships with `hello_controller.js`).
  - `assets/controllers.json` — registry for controllers provided by installed UX packages.
  - `assets/stimulus_bootstrap.js` — starts the Stimulus app and loads controllers; imported by `assets/app.js`.

## Writing a controller

Put files in `assets/controllers/`. The conventional filename is `<name>_controller.js`
(e.g. `hello_controller.js` registers as the `hello` controller). Extend `Controller` from
`@hotwired/stimulus` and export it as the default export:

```javascript
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
    connect() {
        // runs when an element with data-controller="hello" appears
    }
}
```

Any file you drop in `assets/controllers/` is registered automatically — no manual import needed.

TypeScript controllers are supported via `sensiolabs/typescript-bundle` (add `assets/controllers`
to its `source_dir`).

## Activating a controller in markup

Attach a controller with a `data-controller` attribute (the bundle also provides a Twig helper,
covered below):

```html+twig
<div data-controller="hello">...</div>
```

The docs note that **raw `data-*` attributes are the recommended default** because they are
straightforward; reach for the Twig helpers when you need values/classes/outlets escaped and
formatted for you, or when chaining multiple controllers.

## Targets, values, classes, outlets

These are standard Stimulus concepts; the bundle just helps render their attributes.

- **Targets** — `data-<controller>-target="name"`. Multiple names are space-separated.
- **Values** — `data-<controller>-<key>-value="..."`. Keys are kebab-cased; non-scalar values
  (arrays/objects) are JSON-encoded and HTML-escaped.
- **CSS classes** — `data-<controller>-<key>-class="..."`.
- **Outlets** — `data-<controller>-<outlet>-outlet="<css-selector>"`, used to reference other controllers.
- **Action parameters** — `data-<controller>-<key>-param="..."`.

## Twig helpers

Three functions (and matching filters for chaining) emit the attributes. Verified names:
`stimulus_controller`, `stimulus_action`, `stimulus_target`.

`stimulus_controller(name, values = {}, classes = {}, outlets = {})`:

```html+twig
<div {{ stimulus_controller('hello', { name: 'World', data: [1, 2, 3] }) }}>
{# data-controller="hello" data-hello-name-value="World" data-hello-data-value="[1,2,3]" #}
```

Named args let you skip earlier ones, e.g.
`stimulus_controller('hello', controllerClasses: { loading: 'spinner' })` or
`stimulus_controller('hello', controllerOutlets: { other: '.target' })`.

`stimulus_action(controllerName, actionName, eventName = null, parameters = {})`:

```html+twig
<div {{ stimulus_action('controller', 'method') }}>          {# data-action="controller#method" #}
<div {{ stimulus_action('controller', 'method', 'click') }}> {# data-action="click->controller#method" #}
```

Passing `parameters` adds `data-<controller>-<key>-param` attributes alongside the action.

`stimulus_target(controllerName, targetNames = null)` — `targetNames` is a space-separated string:

```html+twig
<div {{ stimulus_target('controller', 'myTarget secondTarget') }}>
{# data-controller-target="myTarget secondTarget" #}
```

**Chaining** — each helper has a matching Twig *filter* to add more controllers/actions/targets
to the same element:

```html+twig
<div {{ stimulus_controller('hello', { name: 'World' })|stimulus_controller('other-controller') }}>
{# data-controller="hello other-controller" data-hello-name-value="World" #}
```

**Forms** — the helper return value is iterable and has a `.toArray()` method, handy for the
`attr` option of `form_start`/`form_row`:

```twig
{{ form_start(form, { attr: stimulus_controller('hello', { name: 'World' }).toArray() }) }}
```

Controller names are normalized to their HTML form: `/` becomes `--`, `_` becomes `-`, and a
leading `@` is stripped — so `@symfony/ux-chartjs/chart` renders as `symfony--ux-chartjs--chart`.

See [references/twig-helpers.md](references/twig-helpers.md) for fuller examples (values + classes +
outlets together, multiple actions, escaping behavior).

## Third-party UX controllers (`controllers.json`)

Installing a UX PHP package (e.g. `symfony/ux-chartjs`) updates `assets/controllers.json` to
register its controller. The bundle activates every enabled controller listed there
automatically. Each entry carries `enabled` and `fetch` keys. Reference the original package name
in the Twig helper (`stimulus_controller('@symfony/ux-chartjs/chart')`) and it is normalized for you.

## Lazy loading

By default every controller (in `assets/controllers/` and `controllers.json`) is downloaded on
every page. For a controller only used on some pages, make it **lazy** so it loads on demand when
a matching element appears:

- Custom controller: add the magic comment `/* stimulusFetch: 'lazy' */` directly above the class.
- Third-party controller: set its `fetch` to `lazy` in `assets/controllers.json`.

(If using TypeScript with StimulusBundle ≤ 2.21.0, ensure `removeComments` is not `true` so the
lazy comment survives compilation.)

## Asset loading: AssetMapper vs Webpack Encore

The plumbing differs by asset system, but the controller code and templates are identical either way.

- **AssetMapper** — `stimulus_bootstrap.js` imports `startStimulusApp` from
  `@symfony/stimulus-bundle`; Flex adds `@symfony/stimulus-bundle` and `@hotwired/stimulus`
  entries to `importmap.php`. The bundle dynamically builds the loader for your controllers and
  enables Stimulus debug mode when the app runs in debug.
- **Webpack Encore** — uses the **Stimulus Bridge**: `webpack.config.js` calls
  `.enableStimulusBridge('./assets/controllers.json')`, and `stimulus_bootstrap.js` imports
  `startStimulusApp` from `@symfony/stimulus-bridge`. The `@hotwired/stimulus` and
  `@symfony/stimulus-bridge` packages are added to `package.json` (run `npm install` and restart Encore).

### Configuration (AssetMapper)

You can override the controller paths in `config/packages/stimulus.yaml` if you don't use the defaults:

```yaml
stimulus:
    controller_paths:
        - '%kernel.project_dir%/assets/controllers'
    controllers_json: '%kernel.project_dir%/assets/controllers.json'
```

## Rules of thumb

- One controller per file under `assets/controllers/`, named `<name>_controller.js`; let the
  bundle auto-register it — don't hand-wire imports in `stimulus_bootstrap.js`.
- Prefer raw `data-*` attributes for simple cases; use the `stimulus_*` Twig helpers when you need
  escaping/JSON-encoding of values, outlets, or chaining.
- Register third-party controllers through `assets/controllers.json`, not by copying their source.
- Make page-specific controllers lazy to avoid loading them everywhere.
- Don't edit `stimulus_bootstrap.js`, `importmap.php`, or `webpack.config.js` by hand beyond what
  the Flex recipe sets up.
