# Admin API client

`createAdminAPIClient<operations>({ ... })` authenticates against the Shopware
Admin API with OAuth. Import admin types from
`@shopware/api-client/admin-api-types` or your generated `adminApiTypes`.

## Auth modes via `credentials`

`credentials` is optional and typed from the `token post /oauth/token` body.
The client authenticates automatically when the session is expired before a
request — you never call `/oauth/token` yourself.

### Password grant (user login)

```typescript
const adminApiClient = createAdminAPIClient<operations>({
  baseURL: `${process.env.SHOP_URL}/api`,
  credentials: {
    grant_type: "password",
    client_id: "administration",
    scope: "write",
    username: process.env.SHOP_ADMIN_USERNAME,
    password: process.env.SHOP_ADMIN_PASSWORD,
  },
});
```

### Client credentials (token-based / scripting)

```typescript
const adminApiClient = createAdminAPIClient<operations>({
  baseURL: `${process.env.SHOP_URL}/api`,
  credentials: {
    grant_type: "client_credentials",
    client_id: "administration",
    client_secret: process.env.SHOP_ADMIN_TOKEN,
  },
});
```

> The OAuth token request uses `scope` (singular), aligned with RFC 6749 /
> League OAuth2.

## Persistent sessions with `sessionData`

`sessionData` seeds the client with a previously stored session
(`{ accessToken, refreshToken, expirationTime }`). Combine it with `credentials`.
Persist updates through the `onAuthChange` hook:

```typescript
export const adminApiClient = createAdminAPIClient<operations>({
  baseURL: "https://your-shop.example/api",
  sessionData: JSON.parse(Cookies.get("sw-admin-session-data") || "{}"),
});

adminApiClient.hook("onAuthChange", (sessionData) => {
  Cookies.set("sw-admin-session-data", JSON.stringify(sessionData), {
    expires: 1,
    path: "/",
    sameSite: "lax",
  });
});
```

If neither `credentials` nor a usable `sessionData`/refresh token is available
when the session expires, the client logs a warning.

## Testing helpers

`getSessionData()` returns the current session; `setSessionData(data)` sets it
in runtime **without** firing `onAuthChange`. Prefer `onAuthChange` in
application code and reserve these for tests.

## Invoke, errors, hooks

`invoke`, `defaultHeaders`, error handling (`ApiClientError`), and hooks behave
as in the Store client. Admin hooks: `onAuthChange`, `onResponseError`,
`onSuccessResponse`, `onDefaultHeaderChanged`. The Admin client sends an
`Authorization: Bearer <token>` header it manages internally — do not set it
manually.
