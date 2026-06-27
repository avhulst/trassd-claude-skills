---
name: ux-toolkit
description: Scaffold ready-to-use, fully-customizable Twig UI components into a Symfony project from a UX kit (Shadcn UI, Flowbite) with the experimental UX Toolkit, via `php bin/console ux:install <recipe>` (optionally `--kit`). Use when bootstrapping a design system, adding prebuilt UI components, or setting up the TailwindCSS + AssetMapper/Encore prerequisites a kit needs. NOTE the Toolkit is EXPERIMENTAL and likely to change.
---

# Symfony UX Toolkit

The UX Toolkit copies ready-to-use, fully-customizable Twig UI components into
your project. A **kit** (e.g. Shadcn UI, Flowbite) is a curated set of these
components, called **recipes** (e.g. `button`, `table`, `badge`). Running the
install command writes the recipe's files **into your codebase** — you own and
edit them like your own code; they are not a runtime dependency you import.

> EXPERIMENTAL: the UX Toolkit is experimental and is likely to change, possibly
> drastically (command names, kit contents, and CSS setup can shift between
> versions). Pin your versions and re-check the official docs before relying on
> it in production.

## Mental model — copied, not depended-on

- A recipe install **scaffolds source files** (Twig components, sometimes CSS/JS)
  into your project. After that, edit them freely — there is no "update from the
  kit" step, so customization is the expected workflow.
- Only **some** components of the upstream design system are ported into each
  kit; do not assume every Shadcn/Flowbite component exists yet.
- Because files are yours after install, re-running the command on existing
  files needs `--force` to overwrite (it warns and skips otherwise).

## Prerequisites: TailwindCSS

Every official kit requires TailwindCSS. Set this up **before** installing
recipes, via one of two stacks:

- **AssetMapper** → install Tailwind through the
  [TailwindBundle](https://symfony.com/bundles/TailwindBundle/current/index.html).
- **Webpack Encore** → follow Tailwind's Symfony framework guide.

You then add the kit's imports/theme to `assets/styles/app.css`. The two stacks
differ only in the import paths (AssetMapper imports from `../vendor/...`; Encore
imports the npm package name). See [references/css-setup.md](references/css-setup.md)
for the per-kit `app.css` snippets and the package-install commands.

## Installing components

The console command is **`ux:install`** (not a `ux:toolkit:*` name). Its argument
is the **recipe** name; the kit is chosen with `--kit` (`-k`).

```bash
# Install a specific recipe; if you omit --kit and the recipe exists in
# several kits, the command asks which kit to use.
php bin/console ux:install button --kit=shadcn

# Interactive: omit the recipe to pick from the kit's available recipes.
php bin/console ux:install --kit=shadcn

# Overwrite files that already exist (default is to warn and skip).
php bin/console ux:install button --kit=shadcn --force
```

Options (verified against the command):

- `recipe` (argument, optional) — the recipe to install, e.g. `button`, `table`.
  Omit it to be prompted interactively from the kit's recipes.
- `--kit` / `-k` — kit name (`shadcn`, `flowbite`) or an external kit URL such as
  `https://github.com/user/my-kit` (optionally `:branch`). Omit it to be prompted
  when the recipe is ambiguous across kits.
- `--destination` / `-d` — target directory (defaults to the current working
  directory).
- `--force` / `-f` — install even if target files already exist.

After installing, the command prints the **installed files** and any **next
steps** — e.g. suggested `composer require` packages or front-end packages to add
via `npm install` or `php bin/console importmap:install`. Run those before using
the component.

## Setting up a kit (recommended order)

1. Choose your asset stack (AssetMapper + TailwindBundle, or Encore + Tailwind).
2. Install the kit's CSS/JS packages and wire up `assets/styles/app.css` (and,
   for Flowbite, `import 'flowbite'` in `assets/app.js`). See
   [references/css-setup.md](references/css-setup.md).
3. Install recipes with `php bin/console ux:install <recipe> --kit=<kit>`.
4. Follow any "Next steps" the command prints.
5. Use the generated `<twig:...>` components in your templates, then customize the
   copied files to taste.

## Inspecting a kit

To inspect a local kit (its recipes, files, and dependencies) use the debug
command:

```bash
php bin/console ux:toolkit:debug-kit ./kits/shadcn
```

## Rules of thumb

- Treat installed components as **your code**: review the generated Twig/CSS,
  then adapt names, classes, and markup to your app.
- Do the **Tailwind + app.css prerequisite first**; recipes assume the kit's
  theme variables are already loaded, so installing components before the CSS
  setup yields unstyled output.
- Use `--kit` explicitly in scripts and docs so installs are reproducible rather
  than relying on interactive prompts.
- Use `--force` deliberately — it overwrites your edits to previously installed
  files.
- Because the Toolkit is experimental, **do not hard-code assumptions** about
  command names or kit contents into automation without pinning versions.
