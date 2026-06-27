# Manifest details, permissions, lifecycle & config

## Meta block

`<meta>` carries identity. `<name>` must match the `custom/apps/<name>` folder.
`<label>` and `<description>` may be repeated with `lang="de-DE"` for
translations. `<author>` and `<copyright>` are **required** — empty or missing
values make `bin/console app:refresh` fail. Other fields: `<version>`,
`<license>`, `<icon>` (path relative to the app root).

## Permissions

Each permission pairs a privilege with an entity:

```xml
<permissions>
    <read>product</read>
    <create>product</create>
    <update>product</update>
    <delete>order</delete>
    <!-- non-CRUD privilege, since 6.4.12.0 -->
    <permission>system:cache:info</permission>
</permissions>
```

CRUD shortcut (since 6.7.3.0) grants read/create/update/delete in one line; use
the individual elements if you must support older versions:

```xml
<crud>product</crud>
```

Users approve the requested permissions during install; API access via the
confirmation credentials is then limited to them. **Read permissions also
govern webhook payload data** — subscribe read access for every entity carried
in a webhook you register.

## Requirements (6.7.10.0+)

```xml
<requirements>
    <public-access/>
</requirements>
```

Each child is an empty marker element; its presence enables the check. Validated
at install/update in `prod` (skipped in `dev`/`test`).

## App lifecycle events

Register webhooks for your app's own lifecycle to react (e.g., purge user data
on removal):

| Event | When |
|-------|------|
| `app.installed` | app installed |
| `app.updated` | app updated |
| `app.deleted` | app removed |
| `app.activated` | inactive app activated |
| `app.deactivated` | active app deactivated |

These arrive as signed `POST` webhooks (same verification as any webhook).

## Admin notifications

`POST /api/notification` with `status` (`success`/`error`/`info`/`warning`) and
`message`; optional `adminOnly` and `requiredPrivileges` restrict who sees it.
Requires the `notification:create` permission. Throttled after 10 requests.

## Configuration via config.xml

Place at `Resources/config/config.xml`. Same schema as plugin configuration.
Stored in `SystemConfig` as `{appName}.config.{fieldName}`.

Read over the API (needs `system_config:read`):

```txt
GET /api/_action/system-config?domain=DemoApp.config&salesChannelId=<id>

{ "DemoApp.config.field1": true, "DemoApp.config.field2": "value" }
```

Write over the API (needs `system_config:create/update/delete`):

```txt
POST /api/_action/system-config?salesChannelId=<id>
Content-Type: application/json

{ "DemoApp.config.field1": true }
```

In Twig: `{{ config('DemoApp.config.field1') }}`. In app scripts the `config`
service exposes `app('field1')` (your app's values, no prefix, no extra
permission) and `get('any.key')` (any value, needs `system_config:read`).
