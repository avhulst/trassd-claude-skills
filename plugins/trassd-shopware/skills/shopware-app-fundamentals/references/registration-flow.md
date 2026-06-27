# Registration & setup flow (handshake)

Registration runs during install for any app whose backend must talk to
Shopware. It proves both sides are authentic and exchanges a per-shop secret
plus Admin API credentials. The request timeout against the app server is 5
seconds.

## Step 1 — Registration request (Shopware → app)

```txt
GET https://my.example.com/registration?shop-id=KIPf0Fz6BUkN&shop-url=http%3A%2F%2Fmy.shop.com&timestamp=159239728
shopware-app-signature: <HMAC-SHA256 of the query string, keyed with app-secret>
sw-version: 6.4.5.0
shopware-shop-signature: <re-registration only: query string signed with the previous shop-secret>
```

Query params: `shop-id`, `shop-url`, `timestamp`. **Verify** by recomputing the
HMAC over the raw query string with the **app-secret** and comparing. For
re-registration also verify `shopware-shop-signature` against the previous
shop-secret — both must match.

The **app-secret** is unique to your app, comes from the Shopware Account at
upload, and never leaves it. For local dev or private (non-store) apps, put a
`<secret>` in `<setup>` instead; such an app cannot be uploaded to the store.

## Step 2 — Registration response (app → Shopware)

```json
{
  "proof": "94b42d39280141de84bd6fc8e538946ccdd182e4558f1e690eabb94f924e7bc7",
  "secret": "random secret string",
  "confirmation_url": "https://my.example.com/registration/confirm"
}
```

- `proof` = HMAC-SHA256 of `shop-id` + `shop-url` + app name, keyed with the
  app-secret. Proves you hold the app-secret.
- `secret` = the **shop-secret** you generate: random, unique per shop, 64–255
  chars. Save it with `shopId` and `shopUrl`; it signs all later shop→app
  traffic.
- `confirmation_url` = where Shopware sends step 3.

To abort a valid-but-rejected install, return `{"error": "The shop URL is invalid"}`.

## Step 3 — Confirmation request (Shopware → app)

If your proof matches the shop-side computation, Shopware `POST`s the
`confirmation_url`:

```json
{
  "apiKey": "SWIARXBSDJRWEMJONFK2OHBNWA",
  "secretKey": "Q1QyaUg3ZHpnZURPeDV3ZkpncXdSRzJpNjdBeWM1WWhWYWd0NE0",
  "timestamp": "1592398983",
  "shopUrl": "http://my.shop.com",
  "shopId": "sqX6cqHi6hbj"
}
```

Body is signed with the **shop-secret** in `shopware-shop-signature` — verify it
before saving. `apiKey`/`secretKey` are the Admin API `client_id`/`client_secret`
for that `shopId` (OAuth client-credentials grant). On re-registration the body
also carries `shopware-shop-signature-previous` (signed with the previous
shop-secret) so you can confirm it is the same installation.

Extra headers (informational): `sw-version` (6.4.1.0+), `sw-context-language`
and `sw-user-language` (6.4.5.0+).

## Secret rotation & shop-url changes

A shop may re-register an existing `shop-id` to rotate the secret or report a
new URL. The re-registration request is signed with both the app-secret and the
previous shop-secret — **validate both**. **Generate a new shop-secret**;
apply the new secret / changed URL only after a valid confirmation. To avoid
breaking in-flight requests, **accept the old secret in parallel for a short
grace period** (e.g., ~1 minute), then reject it.

## Shop migration safeguard (APP_URL changes)

Shopware compares the current `APP_URL` with the one captured when the shopId
was generated. On a mismatch it **stops sending requests** to apps to prevent
data corruption, then lets the user pick a resolver (`bin/console
app:url-change:resolve` or an Admin modal): **MoveShopPermanently** (re-runs
registration, old install stops working), **ReinstallApps** (new shopId; old
install keeps working but its app data is separate), or **UninstallApps**.
Themes without a backend are unaffected.
