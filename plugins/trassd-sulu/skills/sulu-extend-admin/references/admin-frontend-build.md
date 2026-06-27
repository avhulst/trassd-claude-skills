# Building the admin frontend & website assets with Encore

Backing docs: `cookbook/build-admin-frontend.rst`, `cookbook/webpack-encore.rst`.

## Rebuilding the admin SPA

The admin is a React app built with webpack into `public/build/admin`. Rebuild
it after upgrading Sulu or adding custom admin JS.

### Recommended: update-build command

```bash
php bin/adminconsole sulu:admin:update-build
```

Downloads a prebuilt bundle from the `sulu/skeleton` repository when possible.
If you added custom JS it cleans up old build leftovers and builds locally
(Node required).

### Manual build

From the project's admin asset directory:

```bash
cd assets/admin
npm install
npm run build
```

- Supported toolchains (per the `sulu/sulu` test workflow): Node.js 20–25 with
  npm 8–11; pnpm 8–10 or bun 1 as alternatives; no yarn support.
- bun is experimental: `bun run preinstall && bun install && bun run build`.
- Docker variant: run a `node:<version>` container with the project mounted at
  `/var/project`, then build from `/var/project/assets/admin`.

### Build failures

Remove stale lockfiles and `node_modules` across the project before reinstalling:

```bash
rm -rf assets/admin/node_modules vendor/sulu/sulu/node_modules vendor/sulu/sulu/src/Sulu/Bundle/*/Resources/js/node_modules
rm -rf assets/admin/package-lock.json vendor/sulu/sulu/package-lock.json vendor/sulu/sulu/src/Sulu/Bundle/*/Resources/js/package-lock.json
npm cache clean --force   # if still failing
```

## Website assets with Webpack Encore (keep admin separate)

Install the bundle and enable it:

```bash
composer require symfony/webpack-encore-bundle
```

```php
// config/bundles.php
return [
    // ...
    Symfony\WebpackEncoreBundle\WebpackEncoreBundle::class => ['all' => true],
];
```

Sulu already has its own admin JS app, so the default Encore wiring must be
moved out of the way. Create `assets/website/` and move Flex's generated
`assets/*` (e.g. `app.js`, `bootstrap.js`, `controllers/`, `controllers.json`,
`styles/`) into it.

Repoint `webpack.config.js` at a website-specific output and entry:

```diff
 Encore
-    .setOutputPath('public/build/')
+    .setOutputPath('public/build/website/')
-    .setPublicPath('/build')
+    .setPublicPath('/build/website')
-    .addEntry('app', './assets/app.js')
+    .addEntry('app', './assets/website/app.js')
-    .enableStimulusBridge('./assets/controllers.json')
+    .enableStimulusBridge('./assets/website/controllers.json')
```

Match the manifest and Encore output paths:

```diff
 # config/packages/assets.yaml
 framework:
     assets:
-        json_manifest_path: '%kernel.project_dir%/public/build/manifest.json'
+        json_manifest_path: '%kernel.project_dir%/public/build/website/manifest.json'
```

```diff
 # config/packages/webpack_encore.yaml
 webpack_encore:
-    output_path: '%kernel.project_dir%/public/build'
+    output_path: '%kernel.project_dir%/public/build/website'
```

Link the assets in `templates/base.html.twig`:

```twig
{% block style %}{{ encore_entry_link_tags('app') }}{% endblock %}
{% block javascripts %}{{ encore_entry_script_tags('app') }}{% endblock %}
```

Build with `npm install && npm run build`.

> Warning: if a website build ran before moving the output path, it can wipe the
> admin assets. Restore with `git checkout public/build/admin` or rerun
> `bin/console sulu:admin:update-build`.

### Optional: web-js instead of Stimulus

Remove `assets/website/bootstrap.js`, `assets/website/controllers/`, and
`assets/website/controllers.json`; drop `import './bootstrap';` from
`assets/website/app.js`; comment out `.enableStimulusBridge(...)` in
`webpack.config.js`. Then install `web-js` per its repository docs.
