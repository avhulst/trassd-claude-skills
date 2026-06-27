# Complete annotated webspace XML

A full `config/webspaces/website.xml`. The webspace `<key>` matches the filename
(`website`). Every element is optional except `<name>`, `<key>`,
`<localizations>`, `<default-templates>`, and `<portals>`.

```xml
<?xml version="1.0" encoding="utf-8"?>
<webspace xmlns="http://schemas.sulu.io/webspace/webspace"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://schemas.sulu.io/webspace/webspace http://schemas.sulu.io/webspace/webspace-1.1.xsd">

    <!-- Display name in admin -->
    <name>Website</name>
    <!-- Internal key; MUST equal the filename without .xml; unique installation-wide -->
    <key>website</key>

    <!-- Optional: dedicated security system + automatic page access checks -->
    <security permission-check="true">
        <system>website</system>
    </security>

    <!-- Available content locales; nest for fallbacks -->
    <localizations>
        <localization language="en"/>
    </localizations>

    <!-- Default template chosen on new pages, per type -->
    <default-templates>
        <default-template type="page">default</default-template>
        <default-template type="homepage">default</default-template>
    </default-templates>

    <!-- Typed templates Sulu fetches itself (error/error-<code>, search) -->
    <templates>
        <template type="search">search/search</template>
        <template type="error">error/error</template>
    </templates>

    <!-- Optional: hide templates from the page-form dropdown -->
    <excluded-templates>
        <excluded-template>overview</excluded-template>
    </excluded-templates>

    <!-- Navigation contexts editors can assign pages to -->
    <navigation>
        <contexts>
            <context key="main">
                <meta>
                    <title lang="en">Mainnavigation</title>
                </meta>
            </context>
        </contexts>
    </navigation>

    <!-- Optional: visitor-facing content segments; exactly one default="true" -->
    <segments>
        <segment key="w" default="true">
            <meta>
                <title lang="en">Winter</title>
                <title lang="de">Winter</title>
            </meta>
        </segment>
        <segment key="s" default="false">
            <meta>
                <title lang="en">Summer</title>
                <title lang="de">Sommer</title>
            </meta>
        </segment>
    </segments>

    <!-- Optional: page URL strategy (tree_leaf_edit default, or tree_full_edit) -->
    <resource-locator>
        <strategy>tree_leaf_edit</strategy>
    </resource-locator>

    <!-- One or more portals; each key unique installation-wide -->
    <portals>
        <portal>
            <name>Website</name>
            <key>website</key>

            <!-- Environments must match the Symfony environments; omit ports -->
            <environments>
                <environment type="prod">
                    <urls>
                        <url language="en">example.org</url>
                    </urls>
                </environment>
                <environment type="dev">
                    <urls>
                        <url language="en">example.localhost</url>
                    </urls>
                </environment>
            </environments>
        </portal>
    </portals>
</webspace>
```

## Nested localizations with fallbacks

```xml
<localizations>
    <localization language="en">
        <localization language="en" country="us"/>
        <localization language="en" country="gb"/>
    </localization>
    <localization language="de">
        <localization language="de" country="de"/>
        <localization language="de" country="at"/>
        <localization language="de" country="ch"/>
    </localization>
</localizations>
```

Yields seven locales — `en`, `en-us`, `en-gb`, `de`, `de-de`, `de-at`, `de-ch`
— where the country variants fall back to their parent language.

## URL forms

```xml
<!-- Fixed URL bound to one localization (language+country must exist) -->
<url language="de" country="at">www.example.org</url>

<!-- Placeholder URL: expands per matching value -->
<url>www.example.org/{localization}</url>

<!-- Match any host -->
<url>{host}/{localization}</url>
```

Placeholders (example for the `de-at` localization):

| Placeholder | Expands to | Example |
| --- | --- | --- |
| `{localization}` | full localization | `de-at` |
| `{language}` | language only | `de` |
| `{country}` | country (only with `{language}`) | `at` |
