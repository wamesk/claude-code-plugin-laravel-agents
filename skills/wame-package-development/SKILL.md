---
name: wame-package-development
description: "Use when creating a new Laravel composer package or local module and wiring it into a host app. Triggers: 'create a new package/module', 'vytvor nový modul/balík', 'new composer package', 'scaffold a package', 'split this into a module', 'publish a package config/migrations', or 'how does the app discover local packages'. Teaches the clean package layout per laravelpackage.com, ships adapted stub templates, and shows the wildcard path-repository wiring that replaces the old InterNACHI modules:sync command."
---

# WAME package / module development

Create a clean, self-contained composer package (a "module") that follows the
conventions from **https://www.laravelpackage.com**, then wire it into the host
Laravel app so it is auto-discovered. Ships adapted stub templates under `stubs/`.

## When to use

- Creating a **new module/package** in a host app (e.g. `wamesk/example-widget`).
- Extracting existing app code into a reusable package.
- Wiring the host `composer.json` so it **finds local packages** (path repository).
- Publishing a package's **config, migrations, views, or translations**.
- Deciding **register() vs boot()** or how to support new Laravel versions.

## WAME conventions (do not treat as sensitive / do not sanitize)

These are the real, intended standards for WAME packages:

- **Vendor:** `wamesk` (or `wame` for Nova components).
- **PHP namespace root:** `Wame\` — e.g. `Wame\ExampleWidget\`.
- **Auto-discovery:** via `extra.laravel.providers` in the package `composer.json`
  (no manual provider registration in the host app).
- Use a **neutral example** package name such as `example-widget` in docs — never a
  real domain package name.

## Supported versions

Set these constraints in the package `composer.json` (see `stubs/composer.json.stub`).
Everywhere else this skill says "the supported Laravel/Pest version" — use these:

| Dependency        | Constraint                         |
|-------------------|------------------------------------|
| PHP               | `^8.2` (8.2 / 8.3 / 8.4 / 8.5)     |
| `illuminate/*`    | `^10.0 \| ^11.0 \| ^12.0 \| ^13.0` |
| Pest (dev)        | `^3.0 \| ^4.0`                     |

Follow https://www.laravelpackage.com for the authoritative, version-current guide;
this skill only holds the WAME-specific shape and wiring.

## Create a module — step by step

1. **Pick names.** Choose `{{VENDOR}}` (`wamesk`), `{{PACKAGE}}` (`example-widget`),
   `{{NAMESPACE}}` (`Wame`), and `{{CLASS_PREFIX}}` (StudlyCase of the package,
   `ExampleWidget`). The PHP namespace becomes `{{NAMESPACE}}\{{CLASS_PREFIX}}` →
   `Wame\ExampleWidget`.

2. **Scaffold the directory** under the host's package/vendor dir (e.g.
   `wamesk/example-widget/`). See `reference/package-skeleton.md` for the full layout.
   Copy the stubs and replace every `{{PLACEHOLDER}}`:
   - `stubs/composer.json.stub` → `composer.json`
   - `stubs/ServiceProvider.php.stub` → `src/{{CLASS_PREFIX}}ServiceProvider.php`
   - `stubs/ServiceProviderTest.php.stub` → `tests/{{CLASS_PREFIX}}ServiceProviderTest.php`
   - `stubs/migration.php.stub` → `database/migrations/<timestamp>_create_{{TABLE}}_table.php`
   - `stubs/web-routes.php.stub` → `routes/web.php`
   - `stubs/view.blade.php.stub` → `resources/views/example.blade.php`

3. **Wire the host `composer.json`** with a **wildcard path repository** and require
   the package. This is the canonical way the app finds local modules — full details
   in `reference/composer-wiring.md`:
   ```json
   "repositories": [
       { "type": "path", "url": "wamesk/*", "options": { "symlink": true } }
   ],
   "require": {
       "wamesk/example-widget": "*"
   }
   ```

4. **Install & discover.** Run `composer require 'wamesk/example-widget:*'` (or
   `composer update wamesk/example-widget`). Laravel package auto-discovery reads
   `extra.laravel.providers` and registers the ServiceProvider — no manual step,
   and **no `modules:sync` command** (that InterNACHI workflow is replaced by the
   wildcard path repo).

5. **Publish assets** as needed:
   ```bash
   php artisan vendor:publish --tag={{PACKAGE}}-config
   php artisan vendor:publish --tag={{PACKAGE}}-migrations
   php artisan migrate
   ```
   See `reference/service-provider.md` for the publish tags and `loadX` calls.

## Data & defaults — migrations, not seeders

Reference/lookup data and defaults ship via **idempotent `*_seed_*` migrations**
whose `up()` writes rows only when absent — never via database seeders. Factories
stay, for tests only. Static/lookup data may instead use Laravel Sushi. The
`database/seeders/` PSR-4 mapping in the stub exists only for tooling
compatibility; do not put shipped data there. See `reference/package-skeleton.md`.

## Datetime standard

Every table uses `dateTimeTz()` for `created_at` / `updated_at` / `deleted_at` —
never `timestamps()`, `softDeletes()`, or plain `datetime()`. The migration stub
already follows this; keep it that way when you fill it in.

## Reference files

- `reference/package-skeleton.md` — directory layout, every stub & placeholder.
- `reference/composer-wiring.md` — host wildcard path repo, require, PSR-4,
  auto-discovery, and why it replaces `modules:sync`.
- `reference/service-provider.md` — register vs boot, publishing config/
  migrations/assets, loading routes/views/translations, supporting new Laravel
  versions.
