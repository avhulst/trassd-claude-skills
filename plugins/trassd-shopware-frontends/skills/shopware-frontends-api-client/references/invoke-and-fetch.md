# invoke(...) and fetch features

## Operation string format

`invoke` takes an operation key of the form `"<operationName> <method> <path>"`.
The client splits on spaces to obtain the HTTP method and request path, then
fills `{placeholders}` in the path from `pathParams`.

```typescript
const { data, status } = await apiClient.invoke(
  "readNavigation post /navigation/{activeId}/{rootId}",
  {
    headers: { "sw-include-seo-urls": true },
    pathParams: { activeId: "main-navigation", rootId: "main-navigation" },
    body: { depth: 2 },
  },
);
```

Param shape (typed per operation, all optional unless the operation requires
them): `body`, `query`, `pathParams`, `headers`, and `fetchOptions`. The result
is `{ data, status }`.

## Predefining methods

Wrap `invoke` to add a domain-specific layer with defaults:

```typescript
export const readNavigation = (depth: number) =>
  apiClient.invoke("readNavigation post /navigation/{activeId}/{rootId}", {
    pathParams: { activeId: "main-navigation", rootId: "main-navigation" },
    body: { depth },
  });
```

## Fetch features (ofetch)

Per-request options under `fetchOptions`. Available keys: `cache`, `duplex`,
`keepalive`, `priority`, `redirect`, `retry`, `retryDelay`, `retryStatusCodes`,
`signal`, `timeout`. A subset (`retry`, `retryDelay`, `retryStatusCodes`,
`timeout`) can also be set client-wide via `fetchOptions` on the factory.

### AbortController

```typescript
const controller = new AbortController();
const request = apiClient.invoke("readContext get /context", {
  fetchOptions: { signal: controller.signal },
});
controller.abort(); // request rejects with a cancellation error
```

### Timeout

```typescript
apiClient.invoke("readContext get /context", {
  fetchOptions: { timeout: 5000 },
});
```

## Complex criteria via encodeForQuery

Use the separately importable helper for large criteria objects that must ride
in a query string:

```typescript
import { encodeForQuery } from "@shopware/api-client/helpers";

apiClient.invoke("getProducts get /product", {
  query: { _criteria: encodeForQuery(criteria) },
});
```
