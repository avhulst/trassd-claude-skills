# Overrides and patches for @shopware/api-gen

Two ways to correct an inaccurate or outdated OpenAPI spec. Pick based on
whether you want to fix the generated **TypeScript** or the underlying **JSON**.

## TypeScript overrides

Create `api-types/storeApiTypes.overrides.ts` (store) or
`adminApiTypes.overrides.ts` (admin). You must supply **full** definitions for
anything you override — there is no shallow merge of partial members.

```ts
import type { components as mainComponents } from "./storeApiTypes";

export type components = mainComponents & {
  schemas: Schemas;
};

export type Schemas = {
  CustomerAddress: {
    qwe: string;
  };
};

export type operations = {
  "updateCustomerAddress patch /account/address/{addressId}": {
    contentType?: "application/json";
    accept?: "application/json";
    body: {
      city: string;
    };
    response: components["schemas"]["CustomerAddress"];
    responseCode: 200;
  };
};
```

Operation keys follow the `"operationId method /path"` form. An operation may be
a union of variants (different content types / response codes), each a full
object with `body`, `response` and `responseCode`.

## JSON patches (partial overrides)

Patches are applied directly to the JSON schema, so the JSON syntax must be
valid. They are referenced from `api-gen.config.json` under the API-specific
`patches` array. By default the CLI pulls patches from the api-client repo; list
your own files to add to or override them.

A patch file targets components and lists the changes:

```json
{
  "components": {
    "Cart": [
      { "required": ["price"] },
      { "required": ["errors"] }
    ]
  }
}
```

### One patch vs. many

The array form above applies two **independent** patches to `Cart`. The combined
form applies a single patch:

```json
{
  "components": {
    "Cart": {
      "required": ["price", "errors"]
    }
  }
}
```

Prefer multiple independent patches when the same object needs several distinct
corrections that the backend may fix at different times — each becomes stale on
its own, signalling exactly which patch is now safe to delete.

## Config shape

Reference patches and validation rules per API type (the recommended shape):

```json
{
  "$schema": "./node_modules/@shopware/api-gen/api-gen.schema.json",
  "store-api": {
    "rules": ["COMPONENTS_API_ALIAS"],
    "patches": [
      "./node_modules/@shopware/api-client/api-types/storeApiSchema.overrides.json",
      "./api-types/myOwnPatches.overrides.json"
    ]
  },
  "admin-api": {
    "patches": ["adminApiSchema.overrides.json"]
  }
}
```

Root-level `patches` / `rules` still work but are deprecated — migrate to
`store-api` / `admin-api`.
