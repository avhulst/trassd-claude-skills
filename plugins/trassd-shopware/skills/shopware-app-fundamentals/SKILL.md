---
name: shopware-app-fundamentals
description: >-
  Build a Shopware 6 App (the app extension model, distinct from PHP plugins) —
  manifest.xml, app registration handshake & setup, configuration, webhooks, and
  request/response signature verification. Triggers when creating an app, editing
  manifest.xml, or wiring app registration/webhooks.
---

# Shopware 6 App Fundamentals

## Apps vs. plugins

A Shopware **App** is the extension model for Cloud (SaaS) and remote
integrations. Key differences from a PHP **plugin**:

- **No PHP runs in the shop.** An app ships no executable code into the
  Shopware process. Its logic lives on an **external app server** you host.
- **Communicates over HTTP.** Shopware and the app exchange signed HTTP
  requests (registration, webhooks, payment/tax callbacks, etc.).
- **Manifest-driven.** The entire contract is declared in a single
  `manifest.xml` — meta, permissions, webhooks, setup URL, config.
- Apps live in `custom/apps/<AppName>/manifest.xml`. The folder name **must
  match** `<meta><name>`.
- Themes and Admin-UI-only apps need **no backend**; apps that use Admin
  modules, payment methods, tax providers, or webhooks **must implement
  registration**.

Lifecycle (CLI from project root): `bin/console app:refresh` →
`bin/console app:install --activate <AppName>` → `bin/console cache:clear`.
Validate with `bin/console app:validate [<AppName>]`. `<author>` and
`<copyright>` are required or `app:refresh` fails.

## manifest.xml structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/shopware/trunk/src/Core/Framework/App/Manifest/Schema/manifest-3.0.xsd">
    <meta>
        <name>MyExampleApp</name>              <!-- must equal the folder name -->
        <label>My Example App</label>
        <description>A description</description>
        <author>Your Company Ltd.</author>     <!-- required -->
        <copyright>(c) by Your Company Ltd.</copyright>  <!-- required -->
        <version>1.0.0</version>
        <license>MIT</license>
        <icon>Resources/config/plugin.png</icon>
    </meta>
    <setup>
        <registrationUrl>https://my.example.com/registration</registrationUrl>
        <!-- <secret> only for local/private dev; never for store-uploaded apps -->
    </setup>
    <permissions>
        <read>product</read>
        <create>product</create>
        <permission>system:cache:info</permission> <!-- non-CRUD privilege -->
    </permissions>
    <webhooks>
        <webhook name="product-changed"
                 url="https://my.example.com/event/product-changed"
                 event="product.written"/>
    </webhooks>
</manifest>
```

- **`<setup><registrationUrl>`** — where Shopware sends the registration `GET`.
  Required for any app with a backend.
- **`<setup><secret>`** — a hard-coded signing secret for **local development
  or private apps only**. An app with a `<secret>` **cannot be uploaded to the
  store**; store apps get their `app-secret` from the Shopware Account.
- **`<permissions>`** — combine a privilege (`read`/`create`/`update`/`delete`)
  with an entity, or request a non-CRUD privilege via `<permission>`. Read
  permissions also cover data delivered inside webhook payloads, so subscribe
  the entities your webhooks reference. Users approve permissions at install.
- **`<requirements>`** (Shopware 6.7.10.0+) — declare preconditions like
  `<public-access/>`; validated at install in `prod`.

See [references/manifest-and-config.md](references/manifest-and-config.md) for
the CRUD shortcut, app lifecycle webhooks, and notifications.

## Registration handshake (setup)

Registration runs **once at install** so Shopware and your app exchange keys.
Three steps; security-critical at every one.

1. **Registration request** — Shopware sends `GET <registrationUrl>` with query
   params `shop-id`, `shop-url`, `timestamp`. Header `shopware-app-signature`
   carries an HMAC-SHA256 of the **query string**, keyed with your **app-secret**
   (re-registration also sends `shopware-shop-signature` keyed with the old
   shop-secret — validate **both**).
   - **Verify** by recomputing the signature over the full query string with the
     app-secret and comparing with `hash_equals`. Never trust an unverified
     request.

2. **Registration response** — return JSON proving you hold the app-secret and
   handing Shopware a per-shop secret:
   ```json
   {
     "proof": "<HMAC-SHA256( shopId + shopUrl + appName , app-secret )>",
     "secret": "<random shop-secret, 64–255 chars>",
     "confirmation_url": "https://my.example.com/registration/confirm"
   }
   ```
   The `proof` is the HMAC of `shop-id` + `shop-url` + app name. The `secret`
   (the **shop-secret**, distinct from the **app-secret**) must be random,
   unique per shop, and stored alongside `shopId`/`shopUrl`. To reject install,
   return `{"error": "..."}`.

3. **Confirmation request** — if your proof matches, Shopware `POST`s to
   `confirmation_url` with `apiKey`, `secretKey`, `timestamp`, `shopUrl`,
   `shopId`. The body is signed with the **shop-secret** in
   `shopware-shop-signature`.
   - **Verify** this signature against the body before saving. Store `apiKey` /
     `secretKey` as the Admin API `client_id` / `client_secret` for that
     `shopId`.

**app-secret vs shop-secret:** the app-secret is one value for the whole app
(from the Shopware Account, never shared with shops) and signs registration
requests; the shop-secret is generated by your app per shop and signs all later
shop→app traffic (confirmation, webhooks, callbacks). On re-registration
(secret rotation / URL change) accept old + new secret briefly, then invalidate
the old one. See [references/registration-flow.md](references/registration-flow.md).

## Signature verification (HMAC-SHA256)

Every shop→app request after registration is signed; **always verify before
processing**, and **sign your responses** so Shopware can verify them.

- **Incoming verification.** Recompute HMAC-SHA256 with the **shop-secret** and
  compare with `hash_equals`:
  - **GET**: hash the **query string** (strip the `shopware-shop-signature`
    param first); signature in the query.
  - **POST** (webhooks, confirmation): hash the **raw request body**; signature
    in header `shopware-shop-signature`.
  - Do **not** re-parse/re-encode the query — param order and encoding vary by
    language and will break the comparison. Base the HMAC on the *whole* query
    or body; Shopware may add params without it being a breaking change.
- **Outgoing signing.** Sign the response body with HMAC-SHA256 keyed by the
  **shop-secret** and set header `shopware-app-signature`.
- **Replay defense.** Reject requests whose `timestamp` is too old; the
  signature covers it so it can't be forged.

Manual snippets and the App PHP SDK / Symfony Bundle equivalents are in
[references/signature-verification.md](references/signature-verification.md).

## Configuration (config.xml)

Provide user-facing settings via `Resources/config/config.xml` (same schema as
plugin configuration). It renders under **Extensions → My extensions**.

- Values are stored in `SystemConfig` under the key `{appName}.config.{fieldName}`.
- Read/write from the backend over the Admin API:
  `GET /api/_action/system-config?domain={appName}.config` (needs
  `system_config:read`), `POST /api/_action/system-config` (needs
  `system_config:create/update/delete`).
- In Storefront Twig: `{{ config('MyApp.config.fieldName') }}`. In app scripts:
  `services.config.app('fieldName')` (no prefix, no extra permission).

## Webhooks

Subscribe to shop events; Shopware `POST`s JSON to your URL when they fire.

```xml
<webhooks>
    <webhook name="product-written"
             url="https://example.com/event/product-written"
             event="product.written"/>
    <!-- onlyLiveVersion="true" fires only for the live version (6.5.7.0+) -->
</webhooks>
```

Payload shape: `data.event` (the event name, so one endpoint can serve many),
`data.payload` (for `*.written` events only `entity` + `primaryKey` + changed
fields — fetch full data via the API), `source` (`url`, `appVersion`, `shopId`,
`eventId`), and `timestamp`. **Verify the `shopware-shop-signature` header
(HMAC-SHA256 of the body with the shop-secret) before acting** — same mechanism
as the confirmation request. The app needs read permission for any entity that
appears in a subscribed webhook. App lifecycle events (`app.installed`,
`app.updated`, `app.deleted`, `app.activated`, `app.deactivated`) are delivered
the same way. See [references/webhooks.md](references/webhooks.md).
