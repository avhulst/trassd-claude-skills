# Local development & distribution

## Developing a bundle via a local path repository

Before the bundle exists on a public Git remote, install it into a Contao
project from a local directory. In the **project root** `composer.json`:

```json
{
    "repositories": {
        "somevendor/contao-example-bundle": {
            "type": "path",
            "url": "/path/to/your/extension/directory"
        }
    },
    "require": {
        "somevendor/contao-example-bundle": "dev-main"
    }
}
```

`url` may be absolute or relative to the Contao installation. `composer update`
symlinks the path into `vendor/`, so you can develop straight inside
`vendor/somevendor/contao-example-bundle`. `dev-main` is Composer's default
branch alias.

During development, do **not** optimize the autoloader (`-o` /
`--optimize-autoloader`) — new classes won't be picked up until the next dump.
Re-run `composer dump-autoload` without the flag if it was optimized (the Contao
Manager optimizes by default). Re-enable optimization in production.

## Publishing to Git + Packagist

```bash
git init
git add --all
git commit -m "initial commit"
git remote add origin git@github.com:somevendor/contao-example-bundle.git
git push origin main
```

Then submit the repository URL at packagist.org and configure auto-updates (a
GitHub hook). Once on Packagist, the local `path` repository can be removed from
the project root; requiring a `dev-` branch makes Composer check out the Git repo
directly.

### Requirements to appear in the Contao Manager search

All of the following must hold:

- published on packagist.org,
- `"type": "contao-bundle"`,
- at least one **version tag** (branch-only packages are ignored),
- an `extra.contao-manager-plugin` reference in `composer.json`.

Packagist data is cached up to ~12 hours, so listings can take a day to surface.
Add multilingual descriptions / a logo via the `contao/package-metadata`
repository. A package that is not on Packagist or not a `contao-bundle` can still
be listed there with a link to your own homepage.

## Private & commercial distribution

For packages you don't publish publicly:

- **Private Packagist** or a VCS repository — but Composer only reads the
  `repositories` config from the **root** `composer.json`, not from installed
  packages.
- **Artifacts** — a ZIP of the package files (with `composer.json` at the ZIP
  root) uploaded into the Contao Manager. Artifacts **must** declare a `version`
  property in `composer.json`, since Composer can't read Git tags from a ZIP.
  Zip the files individually, not the parent folder.
- **`contao-provider`** — a special artifact type that may carry `repositories`
  entries (no `auth.json` / `config`; put credentials in the repository URL),
  letting it pull in further private packages. Useful e.g. for shipping a license
  key plus a `require`/`repositories` block that installs the licensed software.
