# Styling and extending Tom Select

## Styling

`assets/controllers.json` auto-imports a Tom Select CSS file for basic styling.
For Bootstrap, switch the default stylesheet off and the Bootstrap one on:

```text
"autoimport": {
    "tom-select/dist/css/tom-select.default.css": false,
    "tom-select/dist/css/tom-select.bootstrap5.css": true
}
```

Further customization: override the classes with your own CSS, or use Tom
Select's render templates.

## Passing Tom Select options

Anything Tom Select accepts as a plain (non-function) value can go through the
`tom_select_options` field option:

```php
->add('tags', TextType::class, [
    'autocomplete' => true,
    'tom_select_options' => [
        'create' => true,
        'createOnBlur' => true,
        'delimiter' => ',',
    ],
])
```

A `TextType` like this has no autocompletion but lets users type new "items";
on submit they are sent as a single `delimiter`-joined string. Add
`autocomplete_url` to give it server-backed suggestions.

## Managing plugins

Configure Tom Select plugins under `tom_select_options.plugins`:

```php
'tom_select_options' => [
    'plugins' => [
        'input_autogrow',                                   // enable a plugin
        'dropdown_header' => ['title' => 'Select an ingredient'], // with config
        'clear_button' => false,                            // disable a default plugin
        'remove_button' => false,
    ],
],
```

Plugins that require a JavaScript function as configuration cannot be set this
way — use the JS event approach below instead.

## Extending via JavaScript (custom Stimulus controller)

Some Tom Select options (e.g. `onInitialize`, `onChange`) must be set in JS.
Listen for the two events the core controller dispatches:

```javascript
// assets/controllers/custom-autocomplete_controller.js
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
    initialize() {
        this._onPreConnect = this._onPreConnect.bind(this);
        this._onConnect = this._onConnect.bind(this);
    }

    connect() {
        this.element.addEventListener('autocomplete:pre-connect', this._onPreConnect);
        this.element.addEventListener('autocomplete:connect', this._onConnect);
    }

    disconnect() {
        this.element.removeEventListener('autocomplete:connect', this._onConnect);
        this.element.removeEventListener('autocomplete:pre-connect', this._onPreConnect);
    }

    _onPreConnect(event) {
        // Tom Select not initialized yet — mutate the options here
        event.detail.options.onChange = (value) => { /* ... */ };
    }

    _onConnect(event) {
        // Tom Select is initialized
        event.detail.tomSelect; // the instance
        event.detail.options;   // options used to initialize it
    }
}
```

Load this controller **eagerly** (remove any `stimulusFetch: 'lazy'`) so it can
hear events from the core controller, then attach it alongside the core one:

```php
// in configureOptions() or the EntityType definition
'attr' => ['data-controller' => 'custom-autocomplete'],
```

The full list of controller values (including `tomSelectOptions`) lives in the
package's `controller.ts`.
