# Host app composer wiring

How the host Laravel app discovers a local package. The canonical pattern is a
**wildcard path repository** over the vendor directory that holds your modules.

## 1. Wildcard path repository

Add a single `path` repository whose `url` is a wildcard over the directory that
contains all local packages. This is the cleanest form — you never edit
`repositories` again when you add another module; you just drop a new folder in.

```json
{
    "repositories": [
        {
            "type": "path",
            "url": "wamesk/*",
            "options": { "symlink": true }
        }
    ]
}
```

- `type: path` — Composer resolves the package from the local filesystem.
- `url: "wamesk/*"` — the wildcard matches **every** package folder under the
  `wamesk/` vendor directory (each folder must contain its own `composer.json`).
- `options.symlink: true` — the package is **symlinked** into `vendor/`, so edits
  in the module are picked up live with no reinstall. (Composer falls back to a
  copy on filesystems without symlink support.)

You may register more than one wildcard (e.g. one per vendor dir):

```json
"repositories": [
    { "type": "path", "url": "wamesk/*", "options": { "symlink": true } },
    { "type": "path", "url": "wame/*",   "options": { "symlink": true } }
]
```

> This wildcard form supersedes listing each package with its own explicit
> `{ "type": "path", "url": "./vendor-dir/package" }` entry. One line covers the
> whole directory.

## 2. Require the package

Reference it by its `name` from the package `composer.json`. Local path packages
use the `*` (or `dev-*` branch) constraint:

```json
"require": {
    "wamesk/example-widget": "*"
}
```

Then install / refresh:

```bash
# Quote the constraint: an unquoted `*` is a glob, and zsh aborts with "no matches found".
composer require 'wamesk/example-widget:*'
# or, if already listed:
composer update wamesk/example-widget
```

## 3. PSR-4 stays in the package, not the host

The host app does **not** add the package namespace to its own `autoload.psr-4`.
The package ships its own PSR-4 in its `composer.json` (see `composer.json.stub`),
and Composer wires it through the path repository. The host `autoload` keeps only
its own app namespaces (`App\`, `Database\Factories\`, `Database\Seeders\`).

## 4. Auto-discovery of the service provider

Laravel **package auto-discovery** reads `extra.laravel.providers` from every
installed package's `composer.json` and registers those providers automatically:

```json
"extra": {
    "laravel": {
        "providers": [
            "Wame\\ExampleWidget\\ExampleWidgetServiceProvider"
        ]
    }
}
```

Discovery is (re)built by the `package:discover` script that runs on
`composer install/update` and `dump-autoload`. You never register the provider by
hand in `config/app.php` / `bootstrap/providers.php`.

To opt a package out of discovery, add it to the host's `extra.laravel.dont-discover`.

## 5. No modules:sync — this replaces it

Older module systems (e.g. the InterNACHI `internachi/modular` package) required a
`php artisan modules:sync` command to keep the host `composer.json` in step with
each module's autoload/provider entries.

With the **wildcard path repository + `extra.laravel.providers`** pattern there is
nothing to sync:

- The path wildcard finds the package folder automatically.
- The package's own `composer.json` carries its PSR-4 and provider.
- `composer update` + Laravel auto-discovery wire everything.

So `modules:sync` is not used, and `internachi/modular` is **not a dependency** —
this skill only borrowed and modernized the shape of its scaffolding stubs.

## Quick checklist

- [ ] Package folder lives under the wildcard dir (`wamesk/example-widget/`).
- [ ] Package `composer.json` has correct `name`, PSR-4, and `extra.laravel.providers`.
- [ ] Host `repositories` has the `{ type: path, url: "wamesk/*", options.symlink: true }` entry.
- [ ] Host `require` lists `wamesk/example-widget: "*"`.
- [ ] `composer update wamesk/example-widget` run; provider shows in `php artisan about`.
- [ ] `vendor/wamesk/example-widget` is a symlink to the source folder.
