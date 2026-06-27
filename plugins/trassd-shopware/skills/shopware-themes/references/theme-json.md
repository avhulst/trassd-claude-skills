# theme.json — config field reference

The `config` property holds `tabs`, `blocks`, `sections`, and `fields`. The key of each
field is its **technical name**, also used as the SCSS variable name. Fields show up in
the Theme Manager when `editable: true`.

## Field parameters

| Name | Meaning |
| --- | --- |
| `type` | `color`, `text`, `number`, `fontFamily`, `media`, `checkbox`, `switch`, `url` |
| `value` | Default value |
| `editable` | If `false`, hidden from the Administration |
| `tab` / `block` / `section` | Grouping in the Theme Manager (all optional) |
| `custom` | Arbitrary passthrough data (not processed), available via API |
| `scss` | If `false`, the field is not injected as an SCSS variable |
| `fullWidth` | If `true`, the Administration component renders full width |
| `label` / `helpText` | Inline translation arrays — **deprecated for v6.8**; use Administration snippet keys instead |

Since v6.8, labels/helpText for tabs/blocks/sections/fields are translated through
snippet keys such as
`sw-theme.<technicalName>.<tab>.<block>.<section>.<field>.label`
(and `.helpText`, or `.<index>.label` for field options). `default` is substituted for
any unnamed level.

## Basic color field grouped into tab/block/section

```javascript
{
  "name": "Just another theme",
  "config": {
    "fields": {
      "sw-color-brand-primary": {
        "type": "color",
        "value": "#399",
        "editable": true,
        "tab": "colors",
        "block": "themeColors",
        "section": "importantColors"
      }
    }
  }
}
```

## Text, number, and boolean fields

```javascript
{
  "config": {
    "fields": {
      "modal-padding":    { "type": "text",   "value": "(0, 0, 0, 0)", "editable": true },
      "visible-slides":   { "type": "number", "value": 3, "editable": true,
                            "custom": { "numberType": "int", "min": 1, "max": 6 } },
      "navigation-fixed": { "type": "switch", "value": true, "editable": true }
    }
  }
}
```

`type: "checkbox"` is an alternative to `switch` for boolean values.

## Custom single-select / multi-select

`custom.componentName` selects an Administration component and `custom.options` lists
the choices.

```javascript
{
  "config": {
    "fields": {
      "my-single-select-field": {
        "type": "text",
        "value": "24",
        "editable": true,
        "block": "exampleBlock",
        "section": "exampleSection",
        "custom": {
          "componentName": "sw-single-select",
          "options": [ { "value": "16" }, { "value": "20" }, { "value": "24" } ]
        }
      },
      "my-multi-select-field": {
        "type": "text",
        "editable": true,
        "value": ["green", "blue"],
        "block": "exampleBlock",
        "section": "exampleSection",
        "custom": {
          "componentName": "sw-multi-select",
          "options": [ { "value": "green" }, { "value": "red" }, { "value": "blue" }, { "value": "yellow" } ]
        }
      }
    }
  }
}
```

## Notes

- You always inherit the `Storefront` config and the configs are merged — only provide
  the values you want to change (e.g. `{ "sw-color-brand-primary": { "value": "#00ff00" } }`).
- Overriding a third-party theme's variables is risky: if they rename/remove a variable,
  the theme may fail to compile.
- After editing `theme.json`, run `bin/console theme:refresh`.
