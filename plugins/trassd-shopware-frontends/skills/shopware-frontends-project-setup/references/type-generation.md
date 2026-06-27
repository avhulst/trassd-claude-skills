# Type generation and custom API types

How to generate API types for your Shopware instance and wire them into the API client so
composables and direct `apiClient` calls are type-safe. Distilled from the
`@shopware/nuxt-module` README and the starter templates.

## Why generate types

By default the API client uses the generic Shopware Store API types. Generating types from
your own instance gives you accurate operations and schemas — including commercial
features and project-specific fields — so editor hints and type checks reflect reality.

## Generate

Type generation is a development-only step driven by `@shopware/api-gen`:

```bash
pnpm run generate-types   # runs: shopware-api-gen generate --apiType=store
```

It reads:
- `OPENAPI_JSON_URL` — base URL of your Shopware instance, **without** the `/store-api/`
  suffix.
- `OPENAPI_ACCESS_KEY` — Sales Channel access key used to fetch the OpenAPI schema.
- `api-gen.config.json` — optional config that can list schema override patches, e.g.:

```jsonc
{
  "$schema": "./node_modules/@shopware/api-gen/api-gen.schema.json",
  "store-api": {
    "patches": ["storeApiSchema.overrides.json"]
  }
}
```

Generated type files are written to the project's `api-types/` folder (e.g.
`api-types/storeApiTypes`).

## Register generated types via #shopware

To make the generated types apply everywhere `apiClient` is used, declare them on the
`#shopware` module in a `shopware.d.ts` file at the project root:

```ts
// shopware.d.ts
declare module "#shopware" {
  import type { createAPIClient } from "@shopware/api-client";

  // Default Shopware types (commented out — using local generated types instead):
  // export type operations =
  //   import("@shopware/api-client/store-api-types").operations;
  // export type Schemas =
  //   import("@shopware/api-client/store-api-types").components["schemas"];

  // Locally generated types (in ./api-types):
  export type operations = import("./api-types/storeApiTypes").operations;
  export type Schemas =
    import("./api-types/storeApiTypes").components["schemas"];

  // Your own API client definition built from those operations:
  export type ApiClient = ReturnType<typeof createAPIClient<operations>>;
}
```

The API client instance becomes aware of your custom types through this `#shopware`
declaration, so every `apiClient` usage is typed against your instance. You can also keep
the default types and merge/override only what you need.

## Regenerate after Shopware changes

Re-run `pnpm run generate-types` whenever your Shopware instance's API surface changes
(new fields, plugins, commercial features) to keep the types in sync.
