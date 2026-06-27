# Webhooks

Subscribe to shop events in the manifest; Shopware sends a `POST` to your URL
when each event fires. One endpoint can handle many events (dispatch on
`data.event`).

## Manifest configuration

```xml
<webhooks>
    <webhook name="product-changed"
             url="https://example.com/event/product-changed"
             event="product.written"/>
    <webhook name="order-created"
             url="https://example.com/event/order-created"
             event="order.written"
             onlyLiveVersion="true"/>
</webhooks>
```

Attributes: `name` (your label), `url` (target), `event` (the Shopware event),
and optional `onlyLiveVersion` (6.5.7.0+, default `false`). With
`onlyLiveVersion="true"` the webhook fires only when the entity is written with
the live version id, and the payload is filtered to live-version entries — use
it to ignore drafts (e.g., only act on actually placed orders). It is honored
only for `HookableEntityWrittenEvent` instances; ignored otherwise.

## Payload

```json
{
  "data": {
    "payload": [
      { "entity": "product", "operation": "delete",
        "primaryKey": "7b04ebe416db4ebc93de4d791325e1d9", "updatedFields": [] }
    ],
    "event": "product.written"
  },
  "source": {
    "url": "http://localhost:8000",
    "appVersion": "0.0.1",
    "shopId": "dgrH7nLU6tlE",
    "eventId": "7b04ebe416db4ebc93de4d791325e1d9"
  },
  "timestamp": 123123123
}
```

- `data.event` — the event name (dispatch key).
- `data.payload` — event data. For `*.written` events this is **not** full
  entities (they could be stale); it gives `entity` + `primaryKey` + changed
  fields, so fetch what you need via the shop API. Other events carry the entity
  data but may omit associations.
- `source.url` — the Shopware instance / API base. `source.shopId` — identifies
  the shop. `source.appVersion` — installed app version. `source.eventId` —
  stable across retries (6.4.11.0+); use it for idempotency.
- `timestamp` — when the webhook was handled (6.4.1.0+); use to reject stale
  requests (replay protection).

Informational headers: `sw-version` (6.4.1.0+), `sw-context-language` and
`sw-user-language` (6.4.5.0+).

## Verify before acting

Every webhook carries `shopware-shop-signature` = HMAC-SHA256 of the request
body, keyed with the **shop-secret** assigned to that shop at registration.
Recompute and compare (constant-time) before doing anything — identical
mechanism to the confirmation request. Your app must hold **read** permission
for every entity that appears in a subscribed webhook.

See the Webhook Events Reference in the Shopware docs for the full event list.
