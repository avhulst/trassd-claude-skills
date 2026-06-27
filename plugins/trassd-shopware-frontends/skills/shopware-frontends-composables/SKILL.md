---
name: shopware-frontends-composables
description: Use @shopware/composables for storefront business logic — useProduct, useCart, useUser, useCheckout, useListing, useNavigation and the shopware context. Triggers when building storefront pages/components that read or mutate Shopware 6 state via composables in a Vue 3 / Nuxt 4 headless storefront.
---

# Shopware Frontends composables

`@shopware/composables` is a set of Vue 3 composition functions for building
headless Shopware 6 storefronts. They own state management, UI logic and data
fetching, and talk to the Store-API through the `@shopware/api-client` package.
All guides in the official "building" section are built on top of them.

## Core principle: use composables, never the API client directly

In components and pages, **call composables — do not call `apiClient` directly**.
The composables encapsulate Store-API calls, share state correctly across the
component tree, keep session/context tokens in sync, and are fully typed. Reach
for `useShopwareContext().apiClient` only for endpoints that no composable
covers, and prefer wrapping that in your own composable.

## How the composables work

- **Composition API only.** Call them inside `<script setup>` / a `setup()`
  function so injection works. They return refs / `ComputedRef`s and methods;
  consume the refs reactively in your template.
- **A Shopware context must be installed.** The context is a Vue 3 plugin
  created with `createShopwareContext(app, { devStorefrontUrl })` and registered
  via `app.use(...)`. It needs an `apiClient` provided into the app
  (`app.provide("apiClient", apiClient)`). In Nuxt this is done for you by
  `@shopware/nuxt-module`, which also auto-imports the composables and lets them
  be used as the `@shopware/composables/nuxt-layer` layer.
- **`useShopwareContext()`** injects the context and the `apiClient`; it throws
  a `ContextError` if no context was installed. Most composables read it for
  you, so you rarely call it directly.
- **Shared vs. local state via provide/inject.** State is shared through an
  internal `useContext(injectionName, …)` helper (built on `@vueuse/core`
  `provideLocal` / `injectLocal`). Passing initial data into a composable
  *creates and provides* a new (local) context for that subtree; calling the
  composable without data *injects* the nearest provided context. This is why
  e.g. `useProduct(product)` on a detail page makes the product available to all
  descendant components that call `useProduct()` with no argument — and why
  calling such a composable with no provided context throws a `ContextError`.
- **Session persistence.** The API client is stateless by default. Persist the
  `sw-context-token` via a cookie and the `onContextChanged` hook so sessions
  survive reloads and work in SSR (Nuxt handles this; for custom setups see the
  reference).

## Most important composables

| Composable | Provides |
|---|---|
| `useProduct(product?, configurator?)` | reactive `product`, `configurator` (`PropertyGroup[]`), `changeVariant()` |
| `useCart()` | cart state and mutations; pair with `useAddToCart()`, `useCartItem()`, `useCartNotification()` |
| `useUser()` | authentication & profile — `login`, register, logout, customer data |
| `useSessionContext()` | session/context state — `sessionContext`, `refreshSessionContext()`; underlies currency, language, shipping/billing context |
| `useCheckout()` | checkout flow — shipping/payment methods, place order |
| `useListing(params?)` | product/category listings with filters, sorting, pagination |
| `useCategoryListing()` | injects the listing context created for a category page (provided by a parent; throws if no `createCategoryListingContext` ancestor) |
| `useNavigation(params?)` | `navigationElements` plus `loadNavigationElements()` for the navigation tree |
| `useCmsBlock()` / `useCmsSection()` / `useCmsMeta()` | typed access to the current CMS block / section / page meta when rendering CMS content |

Other commonly used composables include `useProductSearch`,
`useProductSearchSuggest`, `useProductAssociations`, `useProductConfigurator`,
`useProductReviews`, `useCategory`, `useAddress`, `useCustomerOrders`,
`useOrderDetails`, `useWishlist` / `useLocalWishlist` / `useSyncWishlist`,
`useNewsletter`, `useNotifications`, `usePrice`, `useBreadcrumbs`,
`useInternationalization`, `useNavigationContext`, `useUrlResolver`.

## Short examples

Read product state (relies on a provided product context):

```vue
<script setup lang="ts">
import { useProduct, useAddToCart } from "@shopware/composables";

const { product, configurator, changeVariant } = useProduct();
const { addToCart, count } = useAddToCart(product);
</script>

<template>
  <h1>{{ product.translated.name }}</h1>
  <button @click="addToCart()">Add to cart ({{ count }})</button>
</template>
```

Session and login (from the package README):

```vue
<script setup lang="ts">
import { useUser, useSessionContext } from "@shopware/composables";

const { login } = useUser();
const { refreshSessionContext, sessionContext } = useSessionContext();
refreshSessionContext();
</script>
```

Load the navigation tree:

```ts
import { useNavigation } from "@shopware/composables";

const { navigationElements, loadNavigationElements } = useNavigation();
await loadNavigationElements({ depth: 2 });
```

## Rules of thumb

- Build storefront logic on composables; treat the API client as an escape
  hatch and wrap raw calls in your own composable.
- Always run inside a component/page `setup` so injection resolves.
- Ensure the Shopware context is installed before any composable runs
  (automatic with `@shopware/nuxt-module`).
- If a composable that expects a provided context throws `ContextError`, a
  parent must create/provide that context first (e.g. supply the product to
  `useProduct(product)` higher up the tree).
- Generate Store-API types with `@shopware/api-gen` so composable returns are
  fully typed (`Schemas[...]`).

## Reference

- [Context setup & session persistence](references/context-setup.md) —
  installing `createShopwareContext`, providing `apiClient`, and the cookie /
  `onContextChanged` pattern for SSR-safe sessions.
