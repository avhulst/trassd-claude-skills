# Signing & verifying requests/responses

Shopware signs every request it sends to your app server (HMAC-SHA256) so you
can confirm it came from Shopware and was not altered. You sign every response
so Shopware can confirm it came from your app. Verify **before** processing and
sign **all** responses.

Header names: `shopware-app-signature` (registration request from Shopware, and
the signature you put on responses), `shopware-shop-signature` (all later
shop→app requests, keyed with the shop-secret).

## Verifying incoming requests

**Rule:** base the HMAC on the *entire* query string or the *entire* raw body —
never select specific params. Do not re-parse/re-encode the query; param order
and URL encoding vary by language and will break the comparison. Shopware may
add signature params without treating it as a breaking change, so stay flexible.

- **GET requests:** the signature is a query param. Take the raw query string,
  remove the `shopware-shop-signature=...` entry, compute
  `hash_hmac('sha256', queryString, secret)`, and compare with `hash_equals`.
  For the registration request the key is the **app-secret**; for later
  requests it is the **shop-secret**.
- **POST requests** (webhooks, confirmation): the signature is in the
  `shopware-shop-signature` header. Compute
  `hash_hmac('sha256', rawBody, shopSecret)` and compare with `hash_equals`.
  Read the raw body once and rewind the stream so your handler can read it
  again.

Always compare with a constant-time function (`hash_equals`), never `==`.

## Signing outgoing responses

Compute `hash_hmac('sha256', responseBody, shopSecret)` and set it on the
`shopware-app-signature` response header (rewind the body stream after reading
it). Shopware rejects responses without a valid signature.

## Replay protection

Signed payloads include a `timestamp`. Because the signature covers it, an
attacker cannot alter it. Reject requests whose timestamp is too old.

## SDKs

The official **App PHP SDK** (`RequestVerifier` / `DualSignatureRequestVerifier`
for re-registration, `ResponseSigner`) and the **Symfony Bundle** handle
verification, dual-signature re-registration, and response signing
automatically. Prefer them over hand-rolled HMAC when a PHP server is involved.
