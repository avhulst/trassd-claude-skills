# Action buttons — request, responses, app-script target

`<action-button>` elements live in the `<admin>` section of `manifest.xml`. They
add buttons to the smartbar of `detail` and `list` views.

## Declaration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/shopware/trunk/src/Core/Framework/App/Manifest/Schema/manifest-3.0.xsd">
    <meta>
        ...
    </meta>
    <admin>
        <action-button action="setPromotion" entity="promotion" view="detail"
                       url="https://example.com/promotion/set-promotion">
            <label>set Promotion</label>
        </action-button>
        <action-button action="restockProduct" entity="product" view="list"
                       url="https://example.com/restock">
            <label>restock</label>
        </action-button>
    </admin>
</manifest>
```

Attributes:

- `action` — unique identifier for the action; free to choose.
- `entity` — the entity the button operates on.
- `view` — `detail` or `list` (which smartbar to add the button to).
- `url` — target endpoint (absolute, or a relative app-script endpoint).

## Incoming request

On click, the app receives a webhook-like request. The payload includes the
selected `entity`, the `action`, and an array of `ids` (a single id on detail
views). Example body:

```json
{
  "source": { "url": "http://localhost:8000", "appVersion": "1.0.0", "shopId": "F0nWInXj5Xyr" },
  "data":   { "ids": ["2132f284f71f437c9da71863d408882f"], "entity": "product", "action": "restockProduct" },
  "meta":   { "timestamp": 1592403610, "reference": "9e96...", "language": "2fbb..." }
}
```

From Shopware 6.4.1.0 the shop version is sent as a `sw-version` header. Verify
authenticity via `shopware-shop-signature` (SHA256 HMAC of the request body,
signed with the shop secret from registration).

## Response actions (Shopware 6.4.3.0+)

To trigger something in the Administration, respond with a JSON body and a
`shopware-app-signature` header (SHA256 HMAC of the whole response body, signed
with the app secret). An empty body is always valid if no action is needed.

General structure: `actionType` + `payload`.

```json
// notification
{ "actionType": "notification",
  "payload": { "status": "success", "message": "This is the successful message" } }

// reload current page
{ "actionType": "reload", "payload": {} }

// open a new browser tab
{ "actionType": "openNewTab", "payload": { "redirectUrl": "http://google.com" } }

// open a modal with an embedded iframe
{ "actionType": "openModal",
  "payload": { "iframeUrl": "http://google.com", "size": "medium", "expand": true } }
```

Payload fields:

- `actionType` — `notification`, `reload`, `openNewTab`, or `openModal`.
- `redirectUrl` — URL for `openNewTab`.
- `iframeUrl` — embedded link for `openModal`.
- `status` — notification status: `success`, `error`, `info`, `warning`.
- `message` — notification text.
- `size` — modal size: `small`, `medium`, `large`, `fullscreen` (default
  `medium`).
- `expand` — modal expansion: `true`/`false` (default `false`).

From Shopware 6.4.8.0 the `openModal`/`openNewTab` follow-up requests carry the
same query params as module requests (`shop-id`, `shop-url`, `timestamp`,
`sw-context-language`, `sw-user-language`, `shopware-shop-signature`) — verify
the signature.

## App-script endpoint as target (Shopware 6.4.10.0+)

Point the button at a relative custom-endpoint URL instead of an external
server:

```xml
<admin>
    <action-button action="test-button" entity="product" view="list"
                   url="/api/script/action-button">
        <label>test-api-endpoint</label>
    </action-button>
</admin>
```

Handle it in an app script that returns a JSON response with the same
`actionType`/`payload` contract:

```twig
{# Resources/scripts/api-action-button/action-button-script.twig #}
{% set ids = hook.request.ids %}

{% set response = services.response.json({
    "actionType": "notification",
    "payload": { "status": "success", "message": "You selected " ~ ids|length ~ " products." }
}) %}

{% do hook.setResponse(response) %}
```
