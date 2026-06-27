---
name: ux-frontend-frameworks
description: Render React or Vue.js components from Symfony Twig templates with the UX React and UX Vue bridges. Use when embedding a React or Vue component inside a Twig template via react_component() / vue_component(), passing props from PHP, wiring the matching loader controller in assets/app.js, handling Vue/React lifecycle events, or choosing between WebpackEncore and AssetMapper.
---

# Symfony UX: React & Vue frontend frameworks

Symfony UX React (`symfony/ux-react`) and Symfony UX Vue.js (`symfony/ux-vue`) let
you render React or Vue components directly from Twig, handling rendering and the
PHP-to-component data transfer. Both are **thin Stimulus-based bridges**: each Twig
helper just emits the data attributes for a Stimulus controller (`@symfony/ux-react/react`
or `@symfony/ux-vue/vue`) that mounts the component on the element. Version support:
**React 18+** and **Vue 3 only**.

## Where component files live

- **React:** put controller components in `assets/react/controllers/` (`.jsx`/`.tsx`,
  also nested subdirectories). The component must be the **`export default`** — the
  default export is what the bridge resolves and mounts.
- **Vue:** put controller components in `assets/vue/controllers/` (`.vue` single-file
  components).

These top-level components are called **controller components** — they are the ones
meant to be rendered from Twig. Nested helper components imported by them do not need
to live there.

## Register the loader (once, in assets/app.js)

The Flex recipe adds this automatically; keep it intact. The registration globs the
controllers directory so any file you add there becomes renderable.

```js
// assets/app.js — React
import { registerReactControllerComponents } from '@symfony/ux-react';
registerReactControllerComponents(require.context('./react/controllers', true, /\.(j|t)sx?$/));
```

```js
// assets/app.js — Vue
import { registerVueControllerComponents } from '@symfony/ux-vue';
registerVueControllerComponents(require.context('./vue/controllers', true, /\.vue$/));
```

## Render in Twig & pass props from PHP

Call the Twig function **inside the element's attributes** (`<div {{ ... }}>`). It
returns the Stimulus data attributes, so it belongs in the tag, not the body. The
second argument is a Twig hash of props passed straight to the component. The
component name is **relative to the controllers directory** (use `Admin/Widget` for
a nested file).

Exact signatures (verified against the bridge Twig extensions):

- **React:** `react_component(name, props = {}, options = {})`
- **Vue:** `vue_component(name, props = {})`

```html+twig
{# React — props become the component's props object #}
<div {{ react_component('Hello', { fullName: app.user.fullName }) }}>
    Loading...
</div>

{# Vue — props map to the component's declared defineProps() #}
<div {{ vue_component('Hello', { name: app.user.fullName }) }}></div>
```

Anything inside the element (e.g. "Loading...") shows until the component mounts.
Pass real PHP values (`app.user.fullName`, controller variables) as prop values —
that is the PHP-to-component data path.

### React-only: `permanent` option

React's third argument takes options. `{ permanent: true }` keeps the component
mounted when its root element is removed from the DOM — useful when the element is
moved or removed-then-readded (e.g. Turbo with `data-turbo-permanent`).

```html+twig
<div {{ react_component('Hello', { fullName: 'Fabien' }, { permanent: true }) }}></div>
```

## Vue lifecycle events

The Vue bridge dispatches DOM events you can listen to in `assets/app.js`. Use
`vue:before-mount` to customize the Vue app instance before mount (add plugins,
global directives, Vue Router, state). `vue:mount` / `vue:unmount` fire after mount
and on unmount. The event `detail` carries `componentName`, `component`, `props`,
and (for `before-mount`) the `app` instance. See
[references/vue-events.md](references/vue-events.md) for handler shapes and a Vue
Router example. (The React bridge does not document equivalent events.)

## Build setup: WebpackEncore (recommended) vs AssetMapper

Both packages **work best with WebpackEncore**. Install with
`composer require symfony/ux-react` (or `symfony/ux-vue`); the Flex recipe wires
`webpack.config.js` (`.enableReactPreset()` / `.enableVueLoader()`) and the loader
in `assets/app.js`. Then add the framework helper packages
(`npm install -D @babel/preset-react --force` for React; `vue-loader` for Vue) and
run `npm run watch`.

**AssetMapper differs because JSX and `.vue` files are not pure JavaScript:**

- **React + AssetMapper** is possible but requires extra steps: compile `.jsx` to
  plain JS with Babel (`@babel/preset-react`), point the bundle at the built
  directory via `react.controllers_path` in `config/packages/react.yaml`, and import
  sibling components using the `.js` extension (even when the source file is `.jsx`).
- **Vue + AssetMapper** cannot use the `.vue` SFC format at all (it needs a bundler
  like Webpack Encore or Vite). To use Vue with AssetMapper, author components as
  plain `.js` instead of `.vue`.

See [references/build-setup.md](references/build-setup.md) for the AssetMapper config
and import details.

## Rules of thumb

- Put the Twig helper in the element's attribute list (`<tag {{ ... }}>`), never in
  the body.
- React controller components **must** use `export default`; component names are
  relative to `assets/react/controllers` (or `assets/vue/controllers`).
- Keep the `registerReactControllerComponents` / `registerVueControllerComponents`
  call in `assets/app.js` — without it, nothing in the controllers dir renders.
- Prefer WebpackEncore. Only reach for AssetMapper when you accept the JSX-compile
  step (React) or drop `.vue` SFCs entirely (Vue).
- Use `permanent: true` (React) only when the root element legitimately moves or is
  re-added, e.g. under Turbo.
- Pass data as props through the Twig helper — that is the supported PHP-to-component
  channel; don't reach for ad-hoc globals.
