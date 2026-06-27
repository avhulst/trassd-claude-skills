# Blackfire profiling in DDEV

DDEV has built-in Blackfire integration, and the Blackfire CLI is already in the
web container — nothing to install on the host. A Blackfire account is required
(free accounts are no longer offered).

## Credentials

From the Blackfire control panel (Account → Credentials) get four values:
`BLACKFIRE_SERVER_ID`, `BLACKFIRE_SERVER_TOKEN`, `BLACKFIRE_CLIENT_ID`,
`BLACKFIRE_CLIENT_TOKEN`.

Set them as web-environment variables. Globally (easiest):

```bash
ddev config global --web-environment-add="BLACKFIRE_SERVER_ID=<id>,BLACKFIRE_SERVER_TOKEN=<token>,BLACKFIRE_CLIENT_ID=<id>,BLACKFIRE_CLIENT_TOKEN=<token>"
```

Per project, use `ddev config --web-environment-add=...` instead.

You can also edit the config file directly — `web_environment:` in
`$HOME/.ddev/global_config.yaml` (global) or `.ddev/config.yaml` (project):

```yaml
web_environment:
    - OTHER_ENV=something
    - BLACKFIRE_SERVER_ID=dde5f66d-xxxxxx
    - BLACKFIRE_SERVER_TOKEN=09b0ec-xxxxx
    - BLACKFIRE_CLIENT_ID=f5e88b7e-xxxxx
    - BLACKFIRE_CLIENT_TOKEN=00cee15-xxxxx1
```

## Workflow

```bash
ddev start
ddev blackfire on        # off / status to toggle and inspect
```

With Blackfire on, profile via the Chrome/Firefox browser extension, or use the
CLI in the container:

```bash
ddev exec blackfire curl https://<yoursite>.ddev.site
ddev exec blackfire run drush st
ddev ssh   # then use the Blackfire CLI directly
```
