# Xdebug step debugging — IDE setup, ports, troubleshooting

## How the protocol works

Xdebug is a network protocol. The IDE listens on port 9003. When Xdebug is
enabled (`ddev xdebug on`), PHP inside the web container opens a TCP connection
out to the IDE. Docker places the host-side listener at
`host.docker.internal:9003`. The connection must be unblocked end to end;
firewalls and corporate endpoint security are common culprits.

## PhpStorm setup

**Zero-configuration debugging:**

1. Toggle "Start Listening for PHP Debug Connections".
2. Set a breakpoint and visit a page that hits it.
3. PhpStorm asks how to map container paths to your workstation. Check the box
   to map the **whole project directory to `/var/www/html`** — the default may
   leave `vendor/` and other paths unmapped.
4. Under *Run → Edit Configurations*, ensure no stale servers already exist;
   PhpStorm creates one for you if none do.

**Run/Debug configuration:**

1. *Run → Edit configurations*, click *+*, choose *PHP Web Page*.
2. Create a "server" whose *Name* exactly matches your host (e.g.
   `my-site.ddev.site`).
3. Map your repository root → `/var/www/html` as the absolute path on the server.
4. Set a breakpoint and click the debug button.

If you see "Can't find a source position. Server with name
'SITE_NAME.ddev.site' doesn't exist", set the *PHP → Servers* **Name** to
`SITE_NAME.ddev.site` (Name and Host should match).

**Command-line PHP:** `PHP_IDE_CONFIG` is already set inside the web container.
Do a normal web-request debug once first so PhpStorm auto-creates the matching
server, and confirm the top-level project maps to `/var/www/html`.

**additional_hostnames / additional_fqdns:** copy the existing PhpStorm server
(Settings → PHP → Servers) and change host + name to the alternate hostname so
that URL triggers Xdebug too.

## VS Code setup

1. Install the *PHP Debug* extension.
2. *Run → Open Configuration* and add the "Listen for Xdebug" snippet to
   `.vscode/launch.json`.
3. *Terminal → Configure tasks → Create task.json from template → Others* and add
   the "DDEV: Enable/Disable Xdebug" snippets to `.vscode/tasks.json`.
4. Set a breakpoint in `index.php` (it should be solid red; restart if not).
5. *Run → Start Debugging*, select "Listen for Xdebug" — the bottom bar turns
   orange/live. Visit the project and confirm the breakpoint hits.

On Windows WSL2, enable both the *PHP Debug* and *WSL* extensions in your distro.

## Using a port other than 9003

Add an override in `.ddev/php/`, e.g. `.ddev/php/xdebug_client_port.ini` to use
the legacy 9000:

```ini
[PHP]
xdebug.client_port=9000
```

Then point your IDE at the new port. (On PHP < 7.2 you have Xdebug 2.x, so the
key is `xdebug.remote_port`.)

## IDE location overrides (WSL2 / proxy) — unusual

Default `xdebug_ide_location` should be empty. Only set it for special setups:

```bash
ddev config global --xdebug-ide-location=wsl2        # IDE running inside WSL2 / JetBrains Gateway
ddev config global --xdebug-ide-location=container   # IDE proxy inside the web container (e.g. VS Code Language Server)
ddev config global --xdebug-ide-location=""          # reset to default
```

PhpStorm inside WSL2/Linux: add `-Djava.net.preferIPv4Stack=true` via
*Help → Edit Custom VM Options* so it listens on IPv4, not only IPv6.

## Composer

Composer disables Xdebug even when enabled in DDEV. To debug Composer itself,
set `COMPOSER_ALLOW_XDEBUG=1`. Composer may move plugin classes to temp files at
runtime, so use `xdebug_break()` directly in code if breakpoints don't trigger.

## Troubleshooting

Start with the built-in diagnostic tool:

```bash
ddev utility xdebug-diagnose                # checks config, network, IDE settings
ddev utility xdebug-diagnose --interactive  # guided step-by-step that tests your IDE
```

Common checks:

- Reset a non-default `xdebug_ide_location` to `""` and `ddev restart`.
- `ddev logs` showing `Could not connect to debugging client … host.docker.internal:9003`
  usually means a firewall is blocking 9003. Temporarily disable the firewall; if
  that fixes it, re-enable and add a 9003 exception (e.g. `sudo ufw allow 9003`).
  Corporate endpoint security (CrowdStrike, Cisco Secure Endpoint, etc.) can block
  container→host traffic even with the OS firewall off.
- Confirm Xdebug is active in the container: `php -i | grep -i xdebug` should show
  `with Xdebug v3`, and `php -i | grep xdebug.mode` should include `debug`.
- `ddev ssh` then `telnet host.docker.internal 9003` should connect **only** while
  the IDE is listening. If it connects while the IDE is *not* listening, something
  else owns 9003 — find it with `sudo lsof -i :9003 -sTCP:LISTEN`.
- `DDEV_DEBUG=true ddev start` reports how `host.docker.internal` was resolved,
  which helps diagnose WSL2 networking issues.
