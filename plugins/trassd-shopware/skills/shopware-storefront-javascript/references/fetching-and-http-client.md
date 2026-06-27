# Fetching data from a Storefront JS plugin

There are two ways to make network requests from a Storefront plugin: the native
`fetch` API (recommended for new code) and Shopware's `HttpClient` helper.

## Native fetch

`fetch` is the modern, promise-based replacement for `XMLHttpRequest`. Use it to
call Storefront/widget routes or the Store API:

```javascript
// .../src/example-plugin/example-plugin.plugin.js
const { PluginBaseClass } = window;

export default class ExamplePlugin extends PluginBaseClass {
    init() {
        this.fetchData();
    }

    async fetchData() {
        const response = await fetch('/widgets/checkout/info');
        const data = await response.text();   // or response.json()
        console.log(data);
    }
}
```

- `fetch()` resolves to a `Response`; use `response.text()` or `response.json()`
  to read the body.
- Point requests at Storefront routes (e.g. `/widgets/...`) that return the
  markup or JSON you need.

## HttpClient helper

Shopware ships an `HttpClient` service (a thin `XMLHttpRequest` wrapper) for
callback-style requests. It exposes:

- `get(url, callback, contentType = 'application/json')`
- `post(url, data, callback, contentType)`
- `abort()` — cancel the in-flight request.

```javascript
import HttpClient from 'src/service/http-client.service';

const client = new HttpClient();
client.get('/widgets/checkout/info', (response) => {
    // response is the raw response text
});
```

Use whichever fits the surrounding code; native `fetch` is preferred for new
plugins, while `HttpClient` is common in older core plugins and where its
callback/`abort()` semantics are convenient.
