# trassd-tinymce

Skills and agents that enforce **TinyMCE** best practices — the JavaScript
WYSIWYG rich-text editor. Covers initializing and configuring the editor,
driving its instance API, enabling the bundled plugins, authoring custom
plugins with UI, and controlling content filtering & HTML sanitization.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

> Grounded in the TinyMCE open-source editor's public API (its TypeScript API
> definitions and JSDoc). Always consult the official docs at
> [tiny.cloud/docs](https://www.tiny.cloud/docs/) for version-specific details.

## Skills

| Skill | Covers |
|-------|--------|
| `tinymce-integration` | `tinymce.init` / `EditorManager`, selector vs target, classic vs inline, self-host vs CDN, `overrideDefaults`, teardown |
| `tinymce-configuration` | The typed option system: toolbar/menubar, `plugins`, `content_style`/`content_css`, core options |
| `tinymce-editor-api` | Editor instance: get/set/insert content, `execCommand`, events, `UndoManager`, `Formatter` |
| `tinymce-bundled-plugins` | Enabling & configuring the bundled OSS plugins via `plugins` + `PluginManager` |
| `tinymce-custom-plugin` | `PluginManager.add()`, `editor.ui.registry` buttons/menus/icons, custom commands & shortcuts |
| `tinymce-content-filtering` | Schema & `valid_elements`, the parse/serialize pipeline, and XSS-protection options |

## Agents

| Agent | When to use |
|-------|-------------|
| `tinymce-config-reviewer` | Review `tinymce.init` config and custom-plugin/UI code for API correctness (valid options, commands, events, registry usage) and content security (sanitization, `valid_elements`). |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-tinymce@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
