# @shopware/nuxt-module configuration

Reference for configuring the connection and HTTP behavior of a Shopware Frontends Nuxt
storefront. Distilled from the `@shopware/nuxt-module` README and the starter templates.

## What the module does

`@shopware/nuxt-module` sets up a Nuxt project for Shopware Frontends. It:

- bundles and configures `@shopware/api-client` and `@shopware/composables` so they work
  together out of the box,
- registers all composable functions globally (no manual imports),
- shares a single Shopware context across the Nuxt app,
- can also act as a full Nuxt layer for extension scenarios.

The session token is stored in the `sw-context-token` cookie and reused on every request
made by composables or directly through the API client instance.

## Two ways to set the connection

Both are valid. `runtimeConfig` values always override the top-level `shopware` key.

### Top-level `shopware` key

```ts
export default defineNuxtConfig({
  modules: ["@shopware/nuxt-module"],
  shopware: {
    endpoint: "https://your-shop.com/store-api/",
    accessToken: "YOUR_SALES_CHANNEL_ACCESS_TOKEN",
  },
});
```

### Via runtimeConfig.public (template default)

```ts
export default defineNuxtConfig({
  modules: ["@shopware/nuxt-module"],
  runtimeConfig: {
    public: {
      shopware: {
        endpoint: "https://your-shop.com/store-api/",
        accessToken: "YOUR_SALES_CHANNEL_ACCESS_TOKEN",
        devStorefrontUrl: "", // optional, local-dev customer registration
      },
    },
  },
});
```

Prefer the `runtimeConfig.public` form: it is overridable by `NUXT_PUBLIC_*` environment
variables at runtime and is exposed to the client side.

## Config keys

- `endpoint` — the Shopware 6 **Store API** URL (note the `/store-api/` path).
- `accessToken` — the Sales Channel access token (Shopware admin → Settings → Sales
  Channel → API access).
- `devStorefrontUrl` (optional) — storefront domain used during local development so
  customer registration works when `localhost` doesn't match a configured Sales Channel
  domain. Leave empty in production; the storefront then uses `window.location.origin`.

## Default API headers

Set default headers for outgoing API calls via `apiClientConfig.headers`. Server-side and
client-side are configured separately, because plain `runtimeConfig` is server-only while
`runtimeConfig.public` is also available on the client.

```jsonc
{
  "runtimeConfig": {
    // server-side (SSR) headers
    "apiClientConfig": {
      "headers": { "ssr-header-example": "ssr-header-example-value" }
    },
    "public": {
      // client-side headers
      "apiClientConfig": {
        "headers": { "global-header-example": "global-header-example-value" }
      }
    }
  }
}
```

## Using composables once configured

After configuration, composables are globally available with full TypeScript hinting:

```vue
<script setup>
const { login } = useUser();
const { refreshSessionContext } = useSessionContext();
refreshSessionContext();
</script>
```

Direct API access through the shared client:

```vue
<script setup>
const { apiClient } = useShopwareContext();
const apiResponse = await apiClient.invoke(/* params omitted */);
</script>
```

## Overriding bundled package versions

The module ships with specific versions of `@shopware/api-client` and
`@shopware/composables`. To test a different version, install that package directly in
your project's `package.json`; the module will then use yours. Mismatched configurations
may cause unexpected behavior.
