# Build setup details: WebpackEncore vs AssetMapper

Both UX React and UX Vue **work best with WebpackEncore**. AssetMapper is supported
only with caveats, because JSX and `.vue` files are not pure JavaScript.

## WebpackEncore (recommended)

Install the bundle:

```terminal
composer require symfony/ux-react   # or: composer require symfony/ux-vue
```

The Symfony Flex recipe wires things up automatically:

- adds `.enableReactPreset()` (React) or `.enableVueLoader()` (Vue) to `webpack.config.js`
- adds the `registerReactControllerComponents()` / `registerVueControllerComponents()`
  loader call to `assets/app.js`

Then install the framework's helper package and start the watcher:

```terminal
# React
npm install -D @babel/preset-react --force
npm run watch

# Vue
npm install -D vue-loader --force
npm run watch
```

For more complex setups, the JS assets can also be installed via the
`@symfony/ux-react` / `@symfony/ux-vue` npm packages.

## React with AssetMapper

JSX is not pure JavaScript, so extra steps are required:

1. Compile your `.jsx` files to plain JavaScript using Babel and the
   `@babel/preset-react` preset.
2. Point the bundle at the built controllers directory:

   ```yaml
   # config/packages/react.yaml
   react:
       controllers_path: '%kernel.project_dir%/assets/build/react/controllers'
   ```

3. Inside your `.jsx` files, import sibling components using the **`.js`** extension,
   even though the source file is named `.jsx`:

   ```js
   // file is PackageList.jsx, but import it as .js
   import PackageList from '../components/PackageList.js';
   ```

## Vue with AssetMapper

The `.vue` single-file-component format cannot be converted to pure JavaScript
outside a bundler (Webpack Encore or Vite), so **`.vue` files cannot be used with
AssetMapper**.

If you still want Vue with AssetMapper, avoid the `.vue` format and author your
controller components as plain `.js` files instead.
