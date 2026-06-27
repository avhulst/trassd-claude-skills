---
name: shopware-frontends-api-gen
description: >-
  Generate and maintain Shopware API types with @shopware/api-gen — running the
  shopware-api-gen CLI against a Shopware 6 instance, the generated Store/Admin
  type files, and keeping endpoint types in sync. Covers the loadSchema →
  validateJson → generate workflow, .env authentication (OPENAPI_JSON_URL /
  OPENAPI_ACCESS_KEY, admin credentials), overrides/patches via
  api-gen.config.json, and how @shopware/api-client consumes the output. Triggers
  when regenerating API types, adding or fixing endpoint types, configuring the
  generator, or troubleshooting a schema that drifted from the shop.
---

# Shopware Frontends: generating API types with @shopware/api-gen

`@shopware/api-gen` is the CLI that turns a Shopware 6 OpenAPI specification into
TypeScript schemas. The output is consumed by the fully typed
[`@shopware/api-client`](https://www.npmjs.com/package/@shopware/api-client), so
your endpoint calls, request bodies and responses become type-checked.

## Install

Install as a dev dependency in the storefront project:

```sh
pnpm add -D @shopware/api-gen
# or: npm install -D @shopware/api-gen
```

The package exposes two equivalent CLI bins: `shopware-api-gen` and `api-gen`.
You can also invoke it ad-hoc with `pnpx @shopware/api-gen <command>`.

## The core workflow

Types are produced in three ordered steps. Run them in this order:

1. **`loadSchema`** — fetch the OpenAPI JSON from the live shop and save it under
   `api-types/`.
2. **`validateJson`** (optional but recommended) — validate the downloaded JSON
   against the configured ruleset before generating.
3. **`generate`** — transform the JSON into TypeScript. Always run `loadSchema`
   first so `generate` has a fresh schema to read.

```sh
pnpx @shopware/api-gen loadSchema --apiType=store
pnpx @shopware/api-gen validateJson --apiType=store
pnpx @shopware/api-gen generate --apiType=store
```

Every command takes a required `--apiType=store|admin`. `store` and `admin`
schemas are independent — generate each separately for whichever API the
storefront talks to.

Add a project script so regeneration is one command:

```json
{
  "scripts": {
    "generate-types": "shopware-api-gen generate --apiType=store"
  }
}
```

## Authentication (.env)

`loadSchema` reads credentials from a `.env` file in the working directory. It
fails fast and reports the missing variable names if anything required is unset.

**Store API** — needs the shop URL and a store-API access key:

```sh
OPENAPI_JSON_URL="https://your-shop-instance.shopware.store"
OPENAPI_ACCESS_KEY="YOUR_STORE_API_ACCESS_KEY"
```

**Admin API** — needs `OPENAPI_JSON_URL` plus one of two grant types:

```sh
# Option 1 — password grant
SHOPWARE_ADMIN_USERNAME="admin@example.com"
SHOPWARE_ADMIN_PASSWORD="my-password"

# Option 2 — client credentials grant (Admin > Settings > System > Integrations)
SHOPWARE_ADMIN_CLIENT_ID="your-integration-client-id"
SHOPWARE_ADMIN_CLIENT_SECRET="your-integration-secret"
```

When `SHOPWARE_ADMIN_CLIENT_SECRET` (or `..._CLIENT_ID`) is set the client
credentials grant is used automatically; both ID and secret are then required.
Otherwise the password grant is used and both username and password are
required.

## Where the output lands and how it's consumed

- `loadSchema` saves the spec to `api-types/storeApiSchema.json` (or
  `adminApiSchema.json`) — the default filename follows `--apiType`.
- `generate` writes the TypeScript types to `api-types/storeApiTypes.d.ts` (or
  `adminApiTypes.d.ts`).
- `@shopware/api-client` imports those generated `components`, `Schemas` and
  `operations` types, so endpoint paths, bodies and responses are type-checked at
  call sites. Default schema overrides also ship inside the api-client package
  (e.g. `node_modules/@shopware/api-client/api-types/storeApiSchema.overrides.json`).

Do not hand-edit the generated `*.d.ts` files — they are overwritten on every
`generate`. Use overrides/patches (below) to change types durably.

## Keeping types in sync when the shop changes

Whenever the Shopware instance, its API version, or installed plugins change,
the OpenAPI spec changes too. Re-run the full workflow (`loadSchema` →
`generate`) and commit the updated `api-types/*` files. Adding or changing an
endpoint follows the same loop; if the endpoint isn't yet in the live spec, add
it via an override or patch so calls type-check before the backend catches up.

## Adjusting an inaccurate or outdated schema

Two mechanisms, depending on whether you fix the **TypeScript output** or the
**JSON schema**:

- **TS overrides** — add `api-types/storeApiTypes.overrides.ts` (or
  `adminApiTypes.overrides.ts`). You must provide full object definitions for
  any overridden component or operation.
- **JSON patches** — partial overrides applied directly to the JSON schema,
  referenced from `api-gen.config.json`. By default the CLI fetches patches from
  the api-client repository; point it at your own patch files to add or override
  them. Patches are designed to become "stale" once the backend is fixed, so you
  get a signal to remove them.

Prefer the **API-specific** config shape (`store-api` / `admin-api` keys) over
the deprecated root-level `patches` / `rules`:

```json
{
  "$schema": "./node_modules/@shopware/api-gen/api-gen.schema.json",
  "store-api": {
    "rules": ["COMPONENTS_API_ALIAS"],
    "patches": ["storeApiSchema.overrides.json", "./api-types/myStoreApiPatches.json"]
  },
  "admin-api": {
    "rules": ["COMPONENTS_API_ALIAS"],
    "patches": ["adminApiSchema.overrides.json"]
  }
}
```

`rules` are applied per API type: `validateJson --apiType=store` only runs
`store-api.rules`.

See [references/overrides-and-patches.md](references/overrides-and-patches.md)
for the override/patch shapes and the multiple-patch pattern.

## Useful flags & extra commands

- `--debug` — verbose logs plus intermediate files; use when a run misbehaves.
- `--logPatches` — show which patches were applied; use when fixing the schema.
- `split <schemaFile>` (experimental) — break a large schema into per-tag or
  per-path files (`--splitBy=tags|paths`, `--filterBy`, `--list tags|paths`,
  `--outputDir`) so an oversized spec can be imported into Postman/Insomnia.

All four commands (`generate`, `loadSchema`, `validateJson`, `split`) also have a
programmatic API exported from `@shopware/api-gen`. Ensure the required env vars
are present in the Node process when calling them in scripts.

## Rules

- Run `loadSchema` before `generate`; `generate` reads the JSON on disk, it does
  not fetch.
- Always pass `--apiType`; never assume store vs admin.
- Treat `api-types/*.d.ts` as generated artifacts — edit via overrides/patches.
- Use API-specific config (`store-api` / `admin-api`); the root-level
  `patches` / `rules` are deprecated.
- Regenerate and commit types whenever the shop, API version, or plugins change.
