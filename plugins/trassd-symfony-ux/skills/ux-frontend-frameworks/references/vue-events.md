# Vue bridge lifecycle events

The UX Vue bridge dispatches DOM events on the document. Register listeners in
`assets/app.js`. The most common use is customizing the Vue app instance before it
mounts — adding plugins, global directives, shared state, or Vue Router.

## `vue:before-mount`

Fired before a component is mounted. This is the hook for modifying the Vue
application (plugins, global directives, state). The `event.detail` exposes:

- `componentName` — the Vue component's name
- `component` — the resolved Vue component
- `props` — the props that will be injected
- `app` — the Vue application instance (only on this event)

```js
// assets/app.js
document.addEventListener('vue:before-mount', (event) => {
    const { componentName, component, props, app } = event.detail;

    const router = VueRouter.createRouter({
        history: VueRouter.createWebHashHistory(),
        routes: [ /* ... */ ],
    });
    app.use(router);
});
```

## `vue:mount`

Fired after a component has been mounted. `event.detail`: `componentName`,
`component`, `props`.

```js
document.addEventListener('vue:mount', (event) => {
    const { componentName, component, props } = event.detail;
});
```

## `vue:unmount`

Fired after a component has been unmounted. `event.detail`: `componentName`, `props`.

```js
document.addEventListener('vue:unmount', (event) => {
    const { componentName, props } = event.detail;
});
```

## Vue Router history modes

With Vue Router, prefer **hash** or **memory** history mode so the Vue routes are not
served through Symfony controllers.

To use **web** history mode, add a catch-all Symfony route that renders the same
template/Vue component, then prefix all Vue routes with that path:

```php
#[Route('/survey/{path<.+>}')]
public function survey($path = ''): Response
{
    // render the template hosting the Vue component
}
```

```js
const router = VueRouter.createRouter({
    history: VueRouter.createWebHistory(),
    routes: [
        { path: '/survey/list', component: ListSurveys },
        { path: '/survey/create', component: CreateSurvey },
        { path: '/survey/edit/:surveyId', component: EditSurvey },
    ],
});
app.use(router);
```
