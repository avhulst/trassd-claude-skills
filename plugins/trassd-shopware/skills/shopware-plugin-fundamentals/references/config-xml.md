# config.xml — input fields, settings, and components

File location: `src/Resources/config/config.xml`. It renders automatically in
the Administration under Extensions → My extensions → the plugin's Configuration
tab. Always declare the `xsi:noNamespaceSchemaLocation` schema so your IDE can
offer auto-completion and validation.

## Structure rules

- At least one `<card>`, each with a `<title>` and at least one `<input-field>`.
- Every `<input-field>` must start with `<name>`: unique, not translatable, at
  least 4 characters, pattern `[a-zA-Z][a-zA-Z0-9]*`.
- `<title>`, `<label>`, `<placeholder>`, `<helpText>`, and `<option><name>` are
  translatable via the `lang` attribute (default `en-GB`).

## Input field types

Set with the `type` attribute on `<input-field>` (default is `text`):

`text`, `textarea`, `text-editor`, `url`, `password`, `int`, `float`, `bool`
(switch), `checkbox`, `datetime`, `date`, `time`, `colorpicker`,
`single-select`, `multi-select`.

## Common settings

- `<label>`, `<placeholder>`, `<helpText>` — translatable text.
- `<defaultValue>` — imported into the database on install/update; cast to the
  correct PHP type via Symfony's `XmlUtils`.
- `<disabled>` / `<required>` / `<copyable>` — boolean values only
  (`copyable` only for `text` and its extensions).
- `<minLength>` / `<maxLength>` — for `text`, `url`, `password`.
- `<min>` / `<max>` — for `int`, `float`.
- `<options>` with one or more `<option>` (each needs `<id>` + `<name>`) — for
  `single-select` / `multi-select`.

```xml
<input-field type="single-select">
    <name>mailMethod</name>
    <options>
        <option>
            <id>smtp</id>
            <name>English label</name>
            <name lang="de-DE">German label</name>
        </option>
        <option>
            <id>pop3</id>
            <name>English label</name>
        </option>
    </options>
    <defaultValue>smtp</defaultValue>
    <label>Mail method</label>
</input-field>
```

## Advanced components

Use `<component name="...">` to render select Admin components. The component
must declare `<name>` first; remaining child elements are passed as props.
Supported by default: `sw-entity-single-select`, `sw-entity-multi-id-select`,
`sw-media-field`, `sw-text-editor`, `sw-snippet-field`.

```xml
<component name="sw-entity-single-select">
    <name>exampleProduct</name>
    <entity>product</entity>
    <label>Choose a product for the plugin configuration</label>
</component>
```

- `sw-entity-single-select` stores the selected entity ID; `sw-entity-multi-id-select`
  stores an array of IDs.
- For entities without a `name` field, set `<label-property>` (e.g.
  `description` for `mail_template`), otherwise the select renders empty.
- `sw-snippet-field` does not write to the system config — it edits the snippet
  translation for the given `<snippet>` key (available from 6.3.4.0).

## Multi-card example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/shopware/trunk/src/Core/System/SystemConfig/Schema/config.xsd">
    <card>
        <title>Basic Configuration</title>
        <title lang="de-DE">Grundeinstellungen</title>
        <input-field>
            <name>email</name>
            <copyable>true</copyable>
            <label>eMail address</label>
            <placeholder>you@example.com</placeholder>
            <helpText>Please fill in your personal eMail address</helpText>
        </input-field>
    </card>
    <card>
        <title>Advanced Configuration</title>
        <input-field type="password">
            <name>secret</name>
            <label>Secret token</label>
        </input-field>
    </card>
</config>
```
