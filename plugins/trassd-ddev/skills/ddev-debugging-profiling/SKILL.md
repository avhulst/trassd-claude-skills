---
name: ddev-debugging-profiling
description: >-
  Debug and profile inside DDEV — Xdebug step debugging, and Blackfire / Xdebug
  / XHProf profiling. Triggers when enabling step debugging, wiring an IDE to
  DDEV, or profiling a PHP request in a DDEV project.
---

# DDEV Debugging & Profiling

DDEV ships Xdebug, Blackfire, Xdebug-profiler, and XHProf preinstalled in the
web container. None of them are installed on your workstation — you toggle them
from the project directory with `ddev` commands and point your IDE/UI at the
output. Xdebug is **off by default for performance**; turn it on only while you
need it.

## Xdebug step debugging

Xdebug is configured automatically in every project but disabled by default.

```bash
ddev xdebug on       # enable (alias: `ddev xdebug`) — stays on until next start/restart
ddev xdebug off      # disable again for better performance when done
ddev xdebug toggle   # flip current state
ddev xdebug status   # show whether it's on
```

Key facts:

- The IDE is the listener. Your IDE listens on Xdebug's default port **9003**;
  PHP in the container connects out to `host.docker.internal:9003`. PhpStorm and
  VS Code already default to 9003. You may need to open port 9003 in your
  firewall.
- Enabling Xdebug only lasts until the next `ddev start` / `ddev restart`.
- **PhpStorm:** click "Start Listening for PHP Debug Connections", set a
  breakpoint, load a page. PhpStorm offers to create a "server"; map the whole
  project directory to `/var/www/html` in the container. Name the server exactly
  your primary host (e.g. `my-site.ddev.site`). The `PHP_IDE_CONFIG` env var is
  already set in the container, so command-line PHP debugging works too.
- **VS Code:** install the *PHP Debug* extension, add the "Listen for Xdebug"
  config to `.vscode/launch.json` and the enable/disable tasks to
  `.vscode/tasks.json`, set a breakpoint, then *Run → Start Debugging*.
- **Path mapping is essential:** the container path is `/var/www/html` and must
  map to your project root, or breakpoints in `vendor/` and elsewhere won't fire.

Custom port, WSL2/proxy IDE locations, and the `ddev utility xdebug-diagnose`
troubleshooting tool: see [references/step-debugging.md](references/step-debugging.md).

## Xdebug profiling mode

Xdebug can also emit cachegrind profiles instead of step-debugging.

- Create the output directory `.ddev/xdebug`.
- Switch Xdebug to profile mode in `.ddev/php/xdebug.ini`:

  ```ini
  xdebug.mode=profile
  xdebug.start_with_request=yes
  xdebug.output_dir=/var/www/html/.ddev/xdebug
  xdebug.profiler_output_name=trace.%c%p%r%u.out
  ```

- `ddev xdebug on`, make an HTTP request — the profile lands in `.ddev/xdebug`.
- Analyze the cachegrind output in a call-graph viewer such as **kcachegrind**.
- `ddev xdebug off` when done so you stop generating profile files.

## Blackfire profiling

DDEV has built-in Blackfire integration; the Blackfire CLI is in the web container.

1. Create a Blackfire account and grab the Server ID/Token and Client ID/Token
   from your account credentials.
2. Set the four credentials as web-environment variables — easiest globally:

   ```bash
   ddev config global --web-environment-add="BLACKFIRE_SERVER_ID=<id>,BLACKFIRE_SERVER_TOKEN=<token>,BLACKFIRE_CLIENT_ID=<id>,BLACKFIRE_CLIENT_TOKEN=<token>"
   ```

   Use `ddev config --web-environment-add=...` for project-level instead, or edit
   the `web_environment:` key in `$HOME/.ddev/global_config.yaml` (or
   `.ddev/config.yaml`) directly.
3. `ddev start`, then `ddev blackfire on` (`off` / `status` as needed).
4. Profile with the browser extension, or via the CLI:

   ```bash
   ddev exec blackfire curl https://<yoursite>.ddev.site
   ddev exec blackfire run drush st
   ```

Full credential layout and CLI examples: [references/blackfire.md](references/blackfire.md).

## XHProf profiling

DDEV has built-in XHProf with two modes; choose one globally, then restart.

**XHGui mode (recommended, DDEV v1.24.4+):**

```bash
ddev config global --xhprof-mode=xhgui && ddev restart
ddev xhgui on        # start profiling
# visit a few pages to collect data, then:
ddev xhgui launch    # open the XHGui web interface (or `ddev xhgui`)
```

**Traditional `prepend` mode:**

```bash
ddev config global --xhprof-mode=prepend
ddev xhprof on       # alias: `ddev xhprof` / `ddev xhprof enable`; check `ddev xhprof status`
```

- `ddev xhprof on` prints the URL for results; recent runs are at
  `https://<projectname>.ddev.site/xhprof`. Keep that tab open and refresh.
- Hit a page twice to skip first-run cache effects, then drill into functions or
  "View Full Callgraph". Sort columns by run count and inclusive/exclusive wall time.
- Runs are erased on `ddev restart`.

Apache alias requirement and advanced `xhprof_prepend.php` customization:
[references/xhprof.md](references/xhprof.md).
