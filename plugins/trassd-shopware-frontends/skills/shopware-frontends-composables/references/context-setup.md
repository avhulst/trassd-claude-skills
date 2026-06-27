# Shopware context setup & session persistence

In a Nuxt project, `@shopware/nuxt-module` installs the Shopware context,
provides the `apiClient`, auto-imports the composables, and handles the session
cookie for you. The patterns below are for **non-Nuxt setups** (plain Vite/Vue,
Astro islands, etc.) where you wire things up by hand. They are distilled from
`packages/composables/README.md`.

## 1. Create and provide the API client

The client is stateless but accepts an optional `contextToken`. Generate the
Store-API types first with `@shopware/api-gen`, then:

```ts
import { createAPIClient } from "@shopware/api-client";
import type { operations } from "#shopware";

export const apiClient = createAPIClient<operations>({
  baseURL: "https://your-api-instance.com/store-api",
  accessToken: "your-sales-channel-access-token",
});

// provide it into the Vue app so composables can inject it
app.provide("apiClient", apiClient);
```

## 2. Install the Shopware context plugin

```ts
import { createShopwareContext } from "@shopware/composables";

const shopwareContext = createShopwareContext(app, {
  devStorefrontUrl: "https://your-sales-channel-configured-domain.com",
});
app.use(shopwareContext);
```

`useShopwareContext()` then injects both the context and the `apiClient`, and
throws a `ContextError` if either is missing — i.e. if a composable runs before
the context was installed.

## 3. Persist the session across reloads (cookies + SSR)

By default the client is stateless. To keep the cart/login session, persist the
`sw-context-token` in a cookie and rehydrate it on init. Listen to the
`onContextChanged` hook to re-save the token whenever it changes:

```ts
import { createAPIClient } from "@shopware/api-client";
import Cookies from "js-cookie";
import type { operations } from "#shopware";

const shopwareEndpoint = "https://your-shop.example/store-api";

export const apiClient = createAPIClient<operations>({
  baseURL: shopwareEndpoint,
  accessToken: "your-access-token",
  contextToken: Cookies.get("sw-context-token"),
});

apiClient.hook("onContextChanged", (newContextToken) => {
  Cookies.set("sw-context-token", newContextToken, {
    expires: 365, // days
    path: "/",
    sameSite: "lax",
    secure: shopwareEndpoint.startsWith("https://"),
  });
});
```

Because the token lives in the cookie, the same session is reachable during SSR.

## 4. Vite: exclude composables from pre-bundling

```ts
// vite.config.ts
optimizeDeps: {
  exclude: ["@shopware/composables"],
},
```

## Notes

- All composables are fully typed; in Nuxt they are registered globally so type
  hints work without explicit imports.
- Once the context is installed, follow the SKILL guidance: drive components
  through composables and treat `useShopwareContext().apiClient` as an escape
  hatch only.
