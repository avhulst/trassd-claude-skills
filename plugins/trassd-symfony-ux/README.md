# trassd-symfony-ux

Skills and agents that enforce **Symfony UX** best practices — the JavaScript
ecosystem for Symfony built on Stimulus and Hotwired Turbo. Covers writing
Stimulus controllers, building PHP-backed Twig & Live Components, integrating
Turbo, Doctrine-backed autocomplete fields, rendering React/Vue from Twig,
scaffolding UI with the UX Toolkit, and the prebuilt UX bridge packages.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `stimulus-controllers` | StimulusBundle: controllers/targets/values/outlets, the `stimulus_*` Twig helpers, AssetMapper vs Encore wiring |
| `twig-components` | `#[AsTwigComponent]`, props & `mount()`, `#[ExposeInTemplate]`, the attributes bag, slots, anonymous components, `<twig:…>` HTML syntax |
| `live-components` | `#[AsLiveComponent]`, `#[LiveProp]`, `#[LiveAction]`/`#[LiveArg]`, data binding, live validation, events, polling |
| `turbo-integration` | Turbo Drive/Frames/Streams, `turbo_stream()`, `#[Broadcast]` over Mercure |
| `ux-autocomplete` | `#[AsEntityAutocompleteField]`, the `autocomplete` option, securing the results endpoint |
| `ux-frontend-frameworks` | Rendering React & Vue components from Twig (`react_component()` / `vue_component()`), props, loaders, Encore vs AssetMapper |
| `ux-toolkit` | Scaffolding ready-to-use Twig UI components from a kit (Shadcn / Flowbite) via `bin/console ux:install` (experimental) |
| `ux-packages` | The PHP-first bridge packages: Chart.js, Cropper.js, Dropzone, Notify, Icons, Map, Translator, CalendarLink |

## Agents

| Agent | When to use |
|-------|-------------|
| `symfony-ux-reviewer` | Review Stimulus controllers, Twig/Live Components, and Turbo usage against Symfony UX conventions (prop/attribute hygiene, LiveProp writability & security, action/CSRF handling, asset wiring). |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-symfony-ux@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
