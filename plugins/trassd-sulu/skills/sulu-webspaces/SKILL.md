---
name: sulu-webspaces
description: >-
  Configure Sulu webspaces and localization — the webspace XML (portals,
  localizations, URLs, segments, templates) and locale/localization providers.
  Triggers when creating or editing files under config/webspaces, adding or
  nesting locales, defining portal URLs/environments, or customizing locale
  resolution with a custom localization provider or default locale provider.
---

# Sulu Webspaces & Localization

A **webspace** is the top-level content unit in Sulu: each one gets its own
content tree (navigable in the admin). Webspaces are configured as XML files in
the project's `config/webspaces/` directory. A webspace can contain multiple
**portals**, which share content but publish it under different URLs and
localizations.

## Where webspaces live

- One XML file per webspace under `config/webspaces/`.
- **The webspace `<key>` MUST equal the filename without `.xml`** (e.g.
  `config/webspaces/website.xml` → `<key>website</key>`).
- The `<key>` must be unique across all webspaces in the installation; it is
  used internally to generate files and paths.

After adding or changing a webspace, run:

```bash
php bin/adminconsole cache:clear
php bin/websiteconsole cache:clear
php bin/adminconsole sulu:document:initialize
php bin/adminconsole sulu:content:validate:webspaces   # validate on errors
```

When you add a **localization**, reinitialize the page tree instead:

```bash
php bin/adminconsole sulu:page:initialize
```

New webspaces and new localizations have **no permissions** by default — assign
them to roles / in the contact permissions tab, or users won't see them.

## Webspace XML structure

Root element `<webspace>` (namespace `http://schemas.sulu.io/webspace/webspace`,
schema `webspace-1.1.xsd`). Key child elements, in document order:

| Element | Required | Purpose |
| --- | --- | --- |
| `<name>` | yes | Display name shown in the admin. |
| `<key>` | yes | Unique internal key = filename. |
| `<security>` | no | Defines a security `<system>`; `permission-check="true"` enforces page access checks. |
| `<localizations>` | yes | Available content locales (see below). |
| `<default-templates>` | yes | Default page template per type, e.g. `page`, `homepage`. |
| `<templates>` | no | Typed templates Sulu retrieves itself: `error`, `error-<http-code>`, `search`. |
| `<excluded-templates>` | no | Templates hidden from the page-form dropdown. |
| `<navigation>` | no | Navigation `contexts` editors can assign pages to. |
| `<segments>` | no | Visitor-facing content segments (see below). |
| `<resource-locator>` | no | URL `strategy` for pages (see below). |
| `<portals>` | yes | One or more portals with environments/URLs. |
| `<theme>` | no | Theme key (via SuluThemeBundle) for multi-webspace looks. |

Minimal skeleton:

```xml
<webspace xmlns="http://schemas.sulu.io/webspace/webspace"
          xsi:schemaLocation="http://schemas.sulu.io/webspace/webspace http://schemas.sulu.io/webspace/webspace-1.1.xsd">
    <name>Website</name>
    <key>website</key>
    <localizations>
        <localization language="en"/>
    </localizations>
    <default-templates>
        <default-template type="page">default</default-template>
        <default-template type="homepage">default</default-template>
    </default-templates>
    <portals>
        <portal>
            <name>Website</name>
            <key>website</key>
            <environments>
                <environment type="prod">
                    <urls><url language="en">example.org</url></urls>
                </environment>
            </environments>
        </portal>
    </portals>
</webspace>
```

See [references/webspace-xml.md](references/webspace-xml.md) for a complete,
annotated webspace file covering every element.

### Templates

- `<default-templates>`: a `<default-template type="...">` per type (commonly
  `page` and `homepage`) names the template selected by default on new pages.
- `<templates>`: typed templates Sulu fetches programmatically. Sulu uses the
  `error-<http-code>` template (falling back to `error`) for error pages, and
  the `search` template to render search results.
- `<excluded-templates>`: each `<excluded-template>` removes a page template
  from the form dropdown. Optional — pointless with a single webspace.

### Segments (optional)

Segments let one site show different content variants. Each `<segment>` needs a
`key` and a localized `<meta><title>`. **Exactly one segment must be
`default="true"`** — that's what a first-time visitor sees. Visitors switch
segments (stored in a cookie; caching is segment-aware). Pages can then be
assigned a segment in their "Excerpt & Taxonomies" tab; a page with a segment is
hidden in navigation/smart content for visitors on a different segment.

### Resource locator strategy (optional)

`<resource-locator><strategy>...</strategy></resource-locator>` controls how a
page's URL is generated and edited. Both strategies include all ancestors in the
generated locator:

- `tree_leaf_edit` (default): only the last path segment is editable; changing a
  page's locator **also updates child pages'** locators.
- `tree_full_edit`: the whole locator is editable; changing a page's locator
  **does not** update children.

Changing a page locator auto-redirects old URLs to the new URL by default.

## Localizations

List available content locales in `<localizations>`. A localization is a
`language` plus an optional `country`:

```xml
<localization language="de" country="at"/>   <!-- Austrian German -->
```

**Nest** localizations to define fallbacks (nesting has no UI effect, only
fallback behaviour and ghost-page resolution in the column navigation):

```xml
<localizations>
    <localization language="en">
        <localization language="en" country="us"/>
        <localization language="en" country="gb"/>
    </localization>
    <localization language="de">
        <localization language="de" country="de"/>
        <localization language="de" country="at"/>
    </localization>
</localizations>
```

Here `en-us`/`en-gb` fall back to `en`, and `de-de`/`de-at` to `de`. Every
configured localization **must be reachable by a portal URL** — either via a
fixed `language`/`country` `<url>` or a `{localization}` placeholder.

In Twig, build a language switcher from the `localizations` variable (entries
have `locale`, `url`, `country`); the active locale is `app.locale`.

## Portals, environments & URLs

Each `<portal>` has its own `<name>` and `<key>` (key unique installation-wide).
A portal's `<environments>` must match the Symfony environments (`dev`, `stage`,
`prod`); each `<environment type="...">` carries its own `<urls>`.

**Omit the port** from URLs. Each URL must resolve to localization(s) in one of
two ways:

```xml
<!-- Fixed: one URL = one localization (language+country must exist) -->
<url language="de" country="at">www.example.org</url>

<!-- Placeholder: expands to one URL per matching value -->
<url>www.example.org/{localization}</url>
```

Placeholders: `{localization}` (e.g. `de-at`), `{language}` (`de`), `{country}`
(`at`, only meaningful with `{language}`). Use `{host}` to match any host, e.g.
`<url>{host}/{localization}</url>`.

## Custom localization provider

Bundles that ship their own locales should register them so Sulu features (e.g.
security) recognize them. Define a service backed by
`Sulu\Component\Localization\Provider\LocalizationProvider`, pass your locales,
and tag it `sulu.localization_provider`:

```xml
<service id="acme.localization_provider"
         class="Sulu\Component\Localization\Provider\LocalizationProvider">
    <argument>%acme.locales%</argument>
    <tag name="sulu.localization_provider"/>
</service>
```

Custom providers implement `LocalizationProviderInterface`.

## Default locale provider

A **DefaultLocaleProvider** picks the locale when a request lacks one (e.g.
`http://sulu.io` → redirect to `http://sulu.io/en`). Two are built in:

- `sulu_website.default_locale.portal_provider` — uses the portal's default
  localization configuration (default).
- `sulu_website.default_locale.request_provider` — uses the HTTP request's
  preferred language.

Select one via configuration:

```yaml
sulu_website:
    default_locale:
        provider_service_id: sulu_website.default_locale.request_provider
```

A custom provider must implement `DefaultLocaleProviderInterface`.
