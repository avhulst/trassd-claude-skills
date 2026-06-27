---
name: ddev-custom-commands-hooks
description: >-
  Automate DDEV with custom commands and hooks — host/web/db custom commands
  under .ddev/commands, and config.yaml hooks (pre/post start, import, etc.).
  Use when adding a custom `ddev` command (a Bash script under
  .ddev/commands/<host|web|db>/<name>) or when adding/editing a `hooks:` block
  in .ddev/config.yaml (post-start, post-import-db, pre-start, …).
---

# DDEV Custom Commands & Hooks

Two complementary automation mechanisms:

- **Custom commands** — new `ddev <name>` subcommands, defined as Bash scripts.
- **Hooks** — tasks DDEV runs automatically before/after built-in commands,
  declared in `.ddev/config.yaml`.

## Custom commands

A custom command is a Bash script whose **directory decides where it runs**:

- `.ddev/commands/host/<name>` — runs on the **host** machine.
- `.ddev/commands/web/<name>` — runs inside the **web** container.
- `.ddev/commands/db/<name>` — runs inside the **db** container.
- `.ddev/commands/<service>/<name>` — runs in a custom service container
  (that service must mount `ddev-global-cache:/mnt/ddev-global-cache`).
- `$HOME/.ddev/commands/<host|web|db>/<name>` — **global** command, available
  to every project. Global changes require a `ddev start` to be picked up
  (they are copied into the project on start).

Rules:

- The **command name comes from the `## Usage:` annotation line**, not the
  filename. Keep the filename matching the name for clarity.
- The first line should be a shebang: `#!/usr/bin/env bash`.
- Make it executable: `chmod +x .ddev/commands/host/<name>` (also keep LF line
  endings, not CRLF, for container commands).
- Confirm it registered with `ddev -h` (it appears in the command list) or
  `ddev help <name>`.
- Argument and flag handling is plain Bash; DDEV exposes many `DDEV_*`
  environment variables to the script (see
  [references/annotations-and-env.md](references/annotations-and-env.md)).

### Annotation header

Annotations are `##`-prefixed comment lines in the script header that feed the
help output and gate visibility. The common ones:

- `## Description:` — one-line summary in `ddev -h`.
- `## Usage:` — usage string; **its value sets the command name**.
- `## Example:` — example invocation (use `\n` to force a line break).
- `## ExecRaw: true` — (container commands) pass args straight through to the
  container, e.g. so `ddev yarn --help` shows yarn's help, not DDEV's.
  Recommended for most container commands.
- `## HostWorkingDir: true` — (container commands) run from the host's current
  working directory inside the container.
- `## CanRunGlobally: true` — (global host commands) allow running from outside
  any DDEV project directory.

Other supported annotations: `Aliases`, `Flags` (JSON flag definitions),
`AutocompleteTerms`, `ProjectTypes`, `OSTypes` and `HostBinaryExists` (host
only), `DBTypes`, `MutagenSync`. Details and the full `DDEV_*` env list are in
[references/annotations-and-env.md](references/annotations-and-env.md).

### Minimal examples

Host command — `.ddev/commands/host/phpstorm`:

```bash
#!/usr/bin/env bash

## Description: Open PhpStorm with the current project
## Usage: phpstorm
## Example: "ddev phpstorm"
## OSTypes: darwin
## HostBinaryExists: "/Applications/PhpStorm.app"

open -a PhpStorm.app ${DDEV_APPROOT}
```

Container command — `.ddev/commands/web/<name>` (runs inside `web`):

```bash
#!/usr/bin/env bash

## Description: Tail the main solr log
## Usage: solrtail
## Example: ddev solrtail
## ExecRaw: true

tail -f /opt/solr/server/logs/solr.log
```

## Hooks

Hooks run tasks automatically around built-in DDEV commands. Declare them under
the `hooks:` key in `.ddev/config.yaml`: each hook key holds an ordered list of
tasks executed in sequence.

```yaml
hooks:
  post-start:
    - exec: "drush cache:rebuild"
    - exec-host: "echo started"
  post-import-db:
    - exec: "drush user:login"
```

### Common trigger keys

`pre-start` / `post-start`, `pre-import-db` / `post-import-db`,
`pre-import-files` / `post-import-files`, `pre-composer` / `post-composer`,
`pre-share` / `post-share`, `pre-stop` / `post-stop`, plus
`pre-config`/`post-config`, `pre-exec`/`post-exec`, `pre-pull`/`post-pull`,
`pre-push`/`post-push`, and the snapshot hooks
(`pre-snapshot`/`post-snapshot`, `pre-delete-snapshot`/`post-delete-snapshot`,
`pre-restore-snapshot`/`post-restore-snapshot`).

Important: containers are not running yet during `pre-start`, and are stopping
during `post-stop` — so **only `exec-host` tasks work in `pre-start` and
`post-stop`**.

### Task types

- `exec` — run a shell command in a container (defaults to `web`). Optional
  keys: `service:` (e.g. `db`), `user:` (username/UID, e.g. `root`),
  `exec_raw:` (array of args run without shell interpretation).
- `exec-host` — run a command on the host.
- `composer` — run a Composer command in the web container. Supports
  `exec_raw:` as an args array.

Commands requiring user interaction are not supported.

```yaml
hooks:
  pre-import-db:
    - exec: 'mysql -uroot -proot -e "CREATE DATABASE IF NOT EXISTS extra;"'
      service: db
  post-start:
    - composer: install
    - exec:
      exec_raw: [ls, -lR, /var/www/html]
```

Tip: for setup that just installs OS/PHP tooling into a container, a
`.ddev/web-build/Dockerfile` is more efficient than a `post-start` `exec` hook.
