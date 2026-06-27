# Stimulus Twig helpers — extended examples

All examples assume StimulusBundle is installed. The helpers render escaped `data-*` attributes;
controller names are normalized (`/` → `--`, `_` → `-`, leading `@` stripped).

## stimulus_controller — values + classes + outlets together

Full positional signature: `stimulus_controller(name, controllerValues, controllerClasses, controllerOutlets)`.

```html+twig
<div {{ stimulus_controller('hello',
        { name: 'World', data: [1, 2, 3, 4] },
        { loading: 'spinner' },
        { other: '.target' }) }}>
    Hello
</div>
```

Renders (conceptually):

```html
<div
    data-controller="hello"
    data-hello-name-value="World"
    data-hello-data-value="[1,2,3,4]"
    data-hello-loading-class="spinner"
    data-hello-other-outlet=".target">
    Hello
</div>
```

Notes:
- Non-scalar values (arrays/objects) are JSON-encoded; booleans become `true`/`false`.
- All values are HTML-escaped, so `[1,2,3,4]` is emitted as escaped entities that the browser
  decodes back to `[1,2,3,4]`.
- Use named arguments to skip earlier positional ones:

```html+twig
<div {{ stimulus_controller('hello', controllerClasses: { loading: 'spinner' }) }}>
<div {{ stimulus_controller('hello', controllerOutlets: { other: '.target' }) }}>
```

## Chaining controllers

The `stimulus_controller` filter appends another controller to the attributes built so far:

```html+twig
<div {{ stimulus_controller('hello', { name: 'World' })|stimulus_controller('other-controller') }}>
{# data-controller="hello other-controller" data-hello-name-value="World" #}
```

## stimulus_action — events, parameters, chaining

Signature: `stimulus_action(controllerName, actionName, eventName = null, parameters = {})`.

```html+twig
<div {{ stimulus_action('controller', 'method') }}>          {# data-action="controller#method" #}
<div {{ stimulus_action('controller', 'method', 'click') }}> {# data-action="click->controller#method" #}
```

With action parameters:

```html+twig
<div {{ stimulus_action('hello-controller', 'method', 'click', { count: 3 }) }}>
{# data-action="click->hello-controller#method" data-hello-controller-count-param="3" #}
```

Chaining multiple actions on one element via the filter:

```html+twig
<div {{ stimulus_action('controller', 'method')|stimulus_action('other-controller', 'test') }}>
{# data-action="controller#method other-controller#test" #}
```

## stimulus_target — multiple targets and chaining

Signature: `stimulus_target(controllerName, targetNames = null)`; `targetNames` is space-separated.

```html+twig
<div {{ stimulus_target('controller', 'myTarget secondTarget') }}>
{# data-controller-target="myTarget secondTarget" #}

<div {{ stimulus_target('controller', 'myTarget')|stimulus_target('other-controller', 'anotherTarget') }}>
{# data-controller-target="myTarget" data-other-controller-target="anotherTarget" #}
```

## Using helpers with Symfony Forms

Each helper returns an object exposing `.toArray()`, useful for the `attr` option:

```twig
{{ form_start(form, { attr: stimulus_controller('hello', { name: 'World' }).toArray() }) }}

{{ form_row(form.password, { attr: stimulus_action('hello-controller', 'checkPasswordStrength').toArray() }) }}

{{ form_row(form.password, { attr: stimulus_target('hello-controller', 'myTarget').toArray() }) }}
```
