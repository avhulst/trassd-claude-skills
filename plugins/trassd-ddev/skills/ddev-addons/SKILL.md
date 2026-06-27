---
name: ddev-addons
description: >-
  Use and create DDEV add-ons — installing add-ons with `ddev add-on get`,
  managing them, and authoring a custom add-on (install.yaml, files, sections).
  Triggers when installing a DDEV add-on (Redis, Elasticsearch, Solr, Adminer,
  phpMyAdmin, etc.), managing installed add-ons, or building/publishing your own
  add-on.
---

# DDEV Add-ons

DDEV add-ons are pre-packaged extensions that add a service or integration to a
project with a single command. They install into the project's `.ddev`
directory (and optionally the global config dir) and wire themselves into the
project configuration automatically.

Prefer an add-on over hand-rolled Docker Compose when a standard, tested service
exists (Redis, Elasticsearch, Solr, Memcached, …). Use custom compose files only
for specialized services or deep customization.

## Using add-ons

Discover, then install, then restart:

```bash
ddev add-on list                 # all available add-ons
ddev add-on search redis         # filter by name/description (all terms must match)
ddev add-on get ddev/ddev-redis  # install (owner/repo for public repos)
ddev restart                     # apply — required after install/config changes
```

`ddev add-on get` accepts several source forms:

```bash
ddev add-on get ddev/ddev-redis                 # GitHub owner/repo (latest release)
ddev add-on get ddev/ddev-redis --version v1.0.4 # a specific release
ddev add-on get https://github.com/ddev/ddev-drupal-solr/archive/refs/tags/v1.2.3.tar.gz  # release tarball URL
ddev add-on get /path/to/package                # local directory
ddev add-on get /path/to/tarball.tar.gz         # local tarball
ddev add-on get ddev/ddev-redis --project my-project  # target a named project
ddev add-on get ddev/ddev-redis --verbose       # debug a failed install
```

**Official vs third-party.** Officially supported add-ons live under the `ddev/`
org (e.g. `ddev/ddev-redis`, `ddev/ddev-elasticsearch`, `ddev/ddev-solr`,
`ddev/ddev-adminer`, `ddev/ddev-phpmyadmin`). Any public repo carrying the
`ddev-get` GitHub topic is installable the same way; check the add-on's own
README and treat third-party add-ons as community-maintained.

**Private repos** need a GitHub token via `DDEV_GITHUB_TOKEN` (preferred),
`GH_TOKEN`, or `GITHUB_TOKEN`; for other platforms, `git clone` first and
`ddev add-on get` the local path.

### Managing installed add-ons

```bash
ddev add-on list --installed     # what's installed in this project
ddev add-on get ddev/ddev-redis  # re-run to update (preserves your customizations)
ddev add-on remove redis         # remove cleanly by add-on name
```

### Customizing an installed add-on

- **Environment variables (preferred):** many add-ons read a `.ddev/.env.<name>`
  file. Set values with `ddev dotenv set .ddev/.env.redis --redis-tag 7-bookworm`
  then `ddev restart`. Check the add-on's README for supported variables.
- **Compose override:** for deeper changes add a
  `.ddev/docker-compose.<name>_extra.yaml`; this survives add-on updates.
- If you edit a generated add-on file directly, remove its `#ddev-generated`
  line so DDEV won't overwrite it — but override files are preferred.

### Troubleshooting

`ddev describe` (status/URLs), `ddev logs -s <service>` (service logs),
`ddev utility compose-config` (final merged compose), and `ddev restart` after
any config change. "Configuration not applied" almost always means a missing
restart.

## Authoring an add-on

Start from the [`ddev-addon-template`](https://github.com/ddev/ddev-addon-template)
("Use this template"). Every add-on is a repo with an `install.yaml` at its root.

### `install.yaml`

The manifest declares the add-on name, which files to copy, and the action
scripts to run around copying. Core keys:

```yaml
name: my-addon            # name used in `ddev add-on` commands
pre_install_actions: []   # scripts run BEFORE files are copied
project_files: []         # copied into the project's .ddev directory
global_files: []          # copied into the global config directory
post_install_actions: []  # scripts run AFTER files are copied
removal_actions: []        # scripts run on `ddev add-on remove`
```

Advanced keys:

```yaml
ddev_version_constraint: '>= v1.25.2'   # minimum DDEV version
dependencies:                            # auto-installed (same forms as `add-on get`)
  - ddev/ddev-redis
  - https://example.com/addon.tar.gz
yaml_read_files:                         # external YAML exposed to Bash (Go template) actions
  services: ".platform/services.yaml"
```

Dependencies install recursively and are guarded against circular loops; skip
with `ddev add-on get --skip-deps`. `yaml_read_files` exposes values as Go
template variables in **Bash** actions only (not PHP) and is rarely needed.

### Actions: Bash or PHP

Actions are list entries (use `- |` blocks). Each may start with directive
comments: `#ddev-description:` (shown during install) and, for Bash,
`#ddev-warning-exit-code: N` (treat N as a warning) or `#ddev-interactive`
(allow terminal prompts when stdin is a TTY).

- **Bash actions** run on the host — good for permissions, simple file ops,
  package/system commands.
- **PHP actions** run inside a temporary container with the php-yaml extension,
  strict error handling, and access to `$_ENV` DDEV variables (e.g.
  `DDEV_PROJECT`, `DDEV_PROJECT_TYPE`, `DDEV_DOCROOT`, `DDEV_PHP_VERSION`). Good
  for YAML manipulation and conditional, cross-platform logic.

```yaml
post_install_actions:
  - |
    #ddev-description: Configure project settings
    chmod +x .ddev/commands/web/mycommand
```

Put complex logic in separate, namespaced script files (e.g.
`myservice/scripts/setup.php`) listed in `project_files` and `require`d from a
short action. Use `removal_actions` to clean up anything that won't be removed
automatically — files placed outside `.ddev/` or files the user may have edited.

### The `#ddev-generated` marker

Add a `#ddev-generated` comment to **every** file your add-on creates or copies.
DDEV uses it to (1) overwrite the file on a later `ddev add-on get` even if the
user changed it, and (2) delete it on `ddev add-on remove`. Files lacking the
marker are assumed user-modified and left in place — clean those up in
`removal_actions`.

```yaml
# docker-compose.myservice.yaml
#ddev-generated
services:
  myservice:
    image: myimage:latest
```

In PHP generators make it the first line (`"#ddev-generated\n" . yaml_emit(...)`);
in JSON use a `"#ddev-generated": true` property.

### Testing and releasing

- **Bats tests:** the template ships `tests/test.bats` (install from directory,
  install from release, plus your own `health_checks`). Run locally with
  `bats ./tests/test.bats --filter-tags '!release'`. Manual test: `ddev add-on
  get /path/to/your/addon`, verify services, then test removal.
- **Naming:** use the `ddev/ddev-<name>` convention; releases follow semantic
  versioning.
- **Discoverability:** add the `ddev-get` GitHub topic so the add-on shows up in
  `ddev add-on list` / the registry. Run `ddev utility addon-update-checker`
  periodically to stay in sync with the template.
- **Official status:** open an issue on the DDEV repo requesting promotion and
  commit to maintaining the add-on.
