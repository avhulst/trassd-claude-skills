# Translator & CalendarLink

Both bundles are **EXPERIMENTAL** — their APIs may change; pin versions.

## Translator (`symfony/ux-translator`)

Exposes your Symfony translation catalogs to JavaScript with the same `trans()`
mechanism (and ICU Message Format support). Requires StimulusBundle configured.

How it works:

1. On Symfony **cache warm**, your translations are dumped as JavaScript into
   `var/translations/` (plus TypeScript type definitions for autocompletion).
   Refresh them without a full warm with `php bin/console ux:translator:warm-cache`.
2. The Flex recipe creates `assets/translator.js`, which builds a translator and
   re-exports `trans`:

   ```js
   import { createTranslator } from '@symfony/ux-translator';
   import { messages, localeFallbacks } from '../var/translations/index.js';

   const translator = createTranslator({ messages, localeFallbacks });
   export const { trans } = translator;
   ```

3. Use `trans()` in your JS exactly like Symfony's PHP `trans()`:

   ```js
   import { trans } from './translator';

   trans('translation.simple');
   trans('translation.with.parameters', { count: 123, foo: 'bar' });
   trans('translation.multi.domains', {}, 'domain2');          // domain
   trans('translation.multi.locales', {}, 'messages', 'fr');   // locale
   ```

Notes:

- **Do not hand-edit** the generated files in `var/translations/` — they are
  regenerated on cache warm.
- Restrict what gets dumped with `domains` and/or `keys_patterns` (wildcard `*`,
  `!` to exclude) in `config/packages/ux_translator.yaml` to control bundle size.
- Default locale is `en`; set it via the `createTranslator()` argument,
  `setLocale(...)`, or `<html>` attributes. Missing keys return the key unless
  `setThrowWhenNotFound(true)` / `throwWhenNotFound: true` is set.
- In production with AssetMapper, disable the unused TS dump with
  `when@prod: ux_translator: { dump_typescript: false }`.

## CalendarLink (`symfony/ux-calendar-link`)

Generates "Add to calendar" links for Google Calendar, Outlook.com, Office 365,
and iCalendar (`.ics`, used by Apple Calendar, Outlook desktop, Thunderbird).

Build a `CalendarEvent` in PHP and pass it to Twig:

```php
use Symfony\UX\CalendarLink\CalendarEvent;

$event = new CalendarEvent(
    title: 'Symfony Live Paris',
    start: new \DateTimeImmutable('2026-05-14 09:00'),
    end:   new \DateTimeImmutable('2026-05-15 18:00'),
    location: 'Cité Universitaire Paris',
    description: 'Annual Symfony conference in France',
);
```

Render with two Twig functions:

```html+twig
{# one provider — returns an object with .url (and .label) #}
<a href="{{ ux_calendar_link(event, 'google').url }}">Add to Google Calendar</a>

{# all registered providers #}
<ul>
    {% for link in ux_calendar_links(event) %}
        <li><a href="{{ link.url }}">{{ link.label }}</a></li>
    {% endfor %}
</ul>
```

Extra event features:

- **All-day:** `new CalendarEvent(..., allDay: true)`.
- **Recurrence:** `CalendarRecurrence::weekly(count: 10)` — also `minutely()`,
  `daily()`, `monthly()`, `yearly()`, each taking `interval` / `count` / `until`.
- **Reminders:** `CalendarReminder::before(minutes: 15)` — accepts any combo of
  `weeks`/`days`/`hours`/`minutes` (summed) plus an optional `description`.

Provider support is not uniform: reminders (VALARM) are honored only by the
`.ics` output; recurrence (RRULE) works for Google and ICS but not Outlook /
Office 365; `FREQ=MINUTELY` works only in `ics`. Unsupported fields are silently
ignored for that provider.
