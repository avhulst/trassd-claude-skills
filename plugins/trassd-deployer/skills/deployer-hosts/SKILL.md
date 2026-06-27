---
name: deployer-hosts
description: >-
  Define Deployer hosts and their connection config — `host()`, hostname vs
  alias, remote_user, port, deploy_path, host ranges, labels, and a YAML/MAML
  inventory imported with `import()`, keeping SSH secrets in ~/.ssh/config and
  using label selectors to pick hosts. Triggers when defining hosts,
  configuring connections, or editing an inventory file.
---

# Deployer Hosts

A **host** is a deployment target. Define one with `host()`:

```php
host('example.org');
```

Every host carries key-value config and gets two keys for free:

- **`alias`** — the unique identifier inside the recipe (what shows in `[brackets]`
  in output and what a bare selector token matches).
- **`hostname`** — the address used for the SSH connection.

By default both equal the string you passed. Override `hostname` to connect
somewhere other than the alias suggests — the alias stays stable while SSH goes
elsewhere:

```php
host('example.org')
    ->set('hostname', 'example.cloud.google.com');
```

## Connection config

The minimal config for a deploy target is `remote_user` and `deploy_path`:

```php
host('example.org')
    ->set('remote_user', 'deployer')
    ->set('deploy_path', '~/example');
```

Prefer the typed setter methods — they give better IDE autocompletion and are
equivalent to `->set('key', …)`:

```php
host('example.org')
    ->setHostname('example.cloud.google.com')
    ->setRemoteUser('deployer')
    ->setPort(2222)
    ->setDeployPath('~/example');
```

Key connection config keys:

| Key | Description |
|---|---|
| `alias` | Identifier for the host (e.g. `prod`, `staging`). |
| `hostname` | Hostname or IP used for the SSH connection. |
| `remote_user` | SSH username. Defaults to the OS user or `~/.ssh/config`. |
| `port` | SSH port. Default `22`. |
| `config_file` | SSH config file. Default `~/.ssh/config`. |
| `identity_file` | SSH private key file, e.g. `~/.ssh/id_rsa`. |
| `forward_agent` | SSH agent forwarding. Default `true`. |
| `ssh_multiplexing` | SSH multiplexing for performance. Default `true`. |
| `shell` | Shell to use. Default `bash -ls`. |
| `deploy_path` | Deployment directory, e.g. `~/myapp`. |
| `ssh_arguments` | Extra SSH options, e.g. `['-o UserKnownHostsFile=/dev/null']`. |

## Keep SSH secrets out of the recipe

Best practice: keep sensitive SSH parameters — identity keys especially — in
`~/.ssh/config`, not in `deploy.php`. Deployer reads `~/.ssh/config` the same way
the `ssh` command does.

```
Host *
  IdentityFile ~/.ssh/id_rsa
```

The recipe then only names hosts and non-secret connection config; credentials
stay in the developer's (or CI runner's) SSH config.

## Defining many hosts

Share config across several hosts in one call:

```php
host('example.org', 'deployer.org', 'another.org')->setRemoteUser('anton');
```

Expand a **range** into many hosts. Numeric ranges keep leading zeros;
alphabetic ranges work too:

```php
host('www[01:50].example.org'); // www01.example.org … www50.example.org
host('db[a:f].example.org');    // dba.example.org … dbf.example.org
```

Run commands on the local machine with `localhost()` (`run()` then executes
locally); `localhost('ci')` gives it the alias `ci`.

## Labels and selectors

**Labels** are key-value tags for grouping hosts. Use them instead of
hand-listing servers as a fleet grows:

```php
host('admin.example.org')->setLabels(['stage' => 'prod', 'role' => 'web']);
host('web[1:5].example.org')->setLabels(['stage' => 'prod', 'role' => 'web']);
host('db[1:2].example.org')->setLabels(['stage' => 'prod', 'role' => 'db']);
host('test.example.org')->setLabels(['stage' => 'test', 'role' => 'web']);
```

Use `->addLabels([...])` to extend labels on an already-defined host rather than
replacing them.

A **selector** is the recipe's second CLI argument (`dep <task> <selector>`); it
picks hosts by label. Selector syntax:

- `,` — OR (match either group); `&` — AND (same host matches all conditions).
- `|` within a value — OR within that key, e.g. `type=web|db`.
- `!=` — negation, e.g. `type!=web`.

```bash
$ dep deploy 'role=web & stage=prod'      # AND
$ dep deploy 'role=web,role=special'      # OR
$ dep deploy 'type=web|db & env=prod'     # value-OR plus AND
```

Special selectors: `all` (every host) and `alias=…`. A token without `=` is
treated as an alias, so `dep deploy web.example.org` ≡ `dep deploy
alias=web.example.org`. No selector at all prompts you to pick (or auto-selects
when the recipe has a single host).

Set a default selector so plain `dep deploy` targets the right group:

```php
set('default_selector', 'stage=prod&role=web,role=special');
```

From PHP, `select('type=web,env=prod')` returns matching hosts and `on(select(...),
fn)` runs a callback on each; a task can pin a fixed selector with
`->select('type=web|db,env=prod')`.

## YAML / MAML inventory

Move host definitions to a separate inventory file and pull them in with
`import()`. This keeps a PHP recipe focused on tasks while the host list lives in
declarative config (handy for sharing or generating it).

```php title="deploy.php"
import('inventory.yaml');
```

```yaml title="inventory.yaml"
hosts:
  example.org:
    remote_user: deployer
    deploy_path: "~/example"
  deployer.org:
    remote_user: deployer
    deploy_path: "~/deployer"
```

In a YAML/MAML `hosts` block every nested key is forwarded to the host's
setter, so all the standard options (`remote_user`, `deploy_path`, `port`,
`identity_file`, `labels`, `ssh_arguments`, …) work. Labels are a nested object;
set `local: true` to register the entry as a localhost.

```maml
{
  hosts: {
    "prod.example.com": {
      remote_user: "deployer"
      deploy_path: "/var/www/prod"
      labels: { stage: "production" }
    }
  }
}
```

Note that a label and a top-level config key with the same name (e.g. `env`) are
independent — a selector only looks at `labels`.
