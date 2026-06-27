---
name: shopware-frontends-api-client
description: >-
  Use @shopware/api-client to call the Shopware Store API and Admin API —
  createAPIClient / createAdminAPIClient, invoking typed endpoints, headers
  (context token), and error interceptors. Triggers when making Shopware API
  requests from the frontend or wiring the API client (e.g. setting up
  apiClient.ts, calling invoke(...), forwarding sw-context-token, switching
  language/currency, authenticating Admin API, or handling ApiClientError).
---

# Shopware Frontends API Client

`@shopware/api-client` is a dynamic, fully typed HTTP client for Shopware 6,
usable in any JS/TS project. It exposes two factories — `createAPIClient`
(Store API) and `createAdminAPIClient` (Admin API) — plus the `ApiClientError`
class and an `ApiError` type. It is built on [ofetch](https://github.com/unjs/ofetch).

## Store API client

Create one client in a shared module (e.g. `src/apiClient.ts`) and import it
everywhere. Types come from a generic `operations` parameter — use the bundled
`@shopware/api-client/store-api-types` or, recommended, types generated from
your instance with `@shopware/api-gen`.

```typescript
import { createAPIClient } from "@shopware/api-client";
import type { operations } from "./api-types/storeApiTypes";

export const apiClient = createAPIClient<operations>({
  baseURL: "https://your-shop.example/store-api",
  accessToken: "YOUR_SW_ACCESS_KEY", // becomes the sw-access-key header
  contextToken: Cookies.get("sw-context-token"), // optional, restores session
});
```

Rules:
- Pass `operations` as the generic so paths and params are type-checked and
  autocompleted.
- `accessToken` is mapped internally to the `sw-access-key` default header;
  `contextToken` to `sw-context-token`. Do not set those two headers manually.
- Keep the client a singleton; never recreate it per request.

## Typed invoke

Endpoints are addressed by an operation string in the form
`"<operationName> <method> <path>"`. The client splits this string to derive the
HTTP method and request path; params for `body`, `query`, `pathParams`, and
`headers` are typed from the operation.

```typescript
const { data, status } = await apiClient.invoke("readProduct post /product", {
  limit: 2,
});
```

- `invoke(...)` returns `{ data, status }` (typed via `RequestReturnType`); for
  operations that require input the params argument is mandatory, otherwise it
  is optional.
- Path placeholders go in `pathParams` (e.g.
  `"readNavigation post /navigation/{activeId}/{rootId}"` with
  `pathParams: { activeId, rootId }`).
- Per-request fetch features (`signal`, `timeout`, `retry`, `cache`, …) go under
  `fetchOptions`. See [references/invoke-and-fetch.md](references/invoke-and-fetch.md).

## Context token, language & currency headers

The Store API is stateful: the cart/session is identified by `sw-context-token`.
The client reads the incoming `sw-context-token` response header and updates its
default headers automatically; when it changes it fires `onContextChanged`.
Persist the new token (typically in a cookie) so the session survives reloads:

```typescript
apiClient.hook("onContextChanged", (newContextToken) => {
  Cookies.set("sw-context-token", newContextToken, { path: "/", sameSite: "lax" });
});
```

Switch language or currency via the recognized default headers `sw-language-id`
and `sw-currency-id`. Apply them on the client (added to every request) or pass
`headers` on a single `invoke`:

```typescript
apiClient.defaultHeaders.apply({ "sw-language-id": "<language-id>" });
```

Other recognized request headers include `sw-include-seo-urls`, `sw-inheritance`,
and `sw-version-id`. Setting a default header to a falsy value removes it.

## Admin API client

`createAdminAPIClient` works like the Store client but authenticates via OAuth.
Choose an auth mode through `credentials`, and optionally persist tokens with
`sessionData`.

```typescript
import { createAdminAPIClient } from "@shopware/api-client";
import type { operations } from "./api-types/adminApiTypes";

const adminApiClient = createAdminAPIClient<operations>({
  baseURL: `${process.env.SHOP_URL}/api`,
  credentials: {
    grant_type: "password",
    client_id: "administration",
    username: process.env.SHOP_ADMIN_USERNAME,
    password: process.env.SHOP_ADMIN_PASSWORD,
  },
});

await adminApiClient.invoke("...");
```

- `grant_type: "password"` — user login; `grant_type: "client_credentials"`
  with a `client_secret` — token-based / scripting.
- The client auto-(re)authenticates when the session is expired before a
  request; you do not call `/oauth/token` yourself.
- Persist sessions with `sessionData` + the `onAuthChange` hook (stores
  `{ accessToken, refreshToken, expirationTime }`); use `getSessionData` /
  `setSessionData` for tests. Auth details: [references/admin-api.md](references/admin-api.md).

## Error handling

Both clients run an error interceptor on failed responses that throws
`ApiClientError`. Catch it and branch on `instanceof`; `error.details` holds the
raw API response (an object with an `errors` array of `ApiError`), plus
`status`, `statusText`, `url`, and `headers`.

```typescript
import { ApiClientError } from "@shopware/api-client";

try {
  await apiClient.invoke("readProduct post /product", { limit: 2 });
} catch (error) {
  if (error instanceof ApiClientError) {
    console.error(error.details); // raw { errors: [...] } from the API
  } else {
    throw error; // not an API error
  }
}
```

## Hooks

Register listeners with `apiClient.hook(name, cb)` (autocompletes available
names). Store client: `onContextChanged`, `onResponseError`, `onSuccessResponse`,
`onDefaultHeaderChanged`, `onRequest`. Admin client: `onAuthChange`,
`onResponseError`, `onSuccessResponse`, `onDefaultHeaderChanged`. Use
`onResponseError` for centralized logging — it fires before `ApiClientError` is
thrown, so still handle the rejection at the call site.

## Runtime config & helpers

- `apiClient.getBaseConfig()` / `updateBaseConfig({ baseURL?, accessToken? })`
  read or change the endpoint or access key at runtime (e.g. environment
  switch) without recreating the client.
- `encodeForQuery` (from `@shopware/api-client/helpers`) compresses complex
  criteria objects into a base64url string for use in a query parameter such as
  `_criteria`. Import helpers separately to keep the bundle small.

## Type generation & overrides

Generate types from your instance with `@shopware/api-gen` (writes
`api-types/storeApiTypes.ts` / `adminApiTypes.ts`) and point your client and any
`#shopware` declaration at them. For custom fields or spec fixes, use a
`*.overrides.ts` overlay (full object definitions required) or a JSON patch
file. See the bundled README / `@shopware/api-gen` docs for the full reference.
