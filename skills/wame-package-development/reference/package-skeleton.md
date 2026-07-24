# Package skeleton

The directory layout of a clean local module, following
https://www.laravelpackage.com. Neutral example: `wamesk/example-widget`
with PHP namespace `Wame\ExampleWidget`.

## Placeholders

Every stub uses these `{{TOKEN}}` placeholders. Replace them all before use.

| Placeholder        | Meaning                                   | Example                     |
|--------------------|-------------------------------------------|-----------------------------|
| `{{VENDOR}}`       | Composer vendor                           | `wamesk`                    |
| `{{PACKAGE}}`      | Composer package name (kebab-case)        | `example-widget`            |
| `{{NAMESPACE}}`    | PHP namespace root                        | `Wame`                      |
| `{{CLASS_PREFIX}}` | StudlyCase package name / class prefix    | `ExampleWidget`             |
| `{{DESCRIPTION}}`  | One-line package description              | `Reusable widget package.`  |
| `{{MODULE_SLUG}}`  | Route/view slug (kebab-case)              | `example-widget`            |
| `{{TABLE}}`        | Migration table name                      | `example_widgets`           |

> The PHP namespace root (`{{NAMESPACE}}`) is defined per-project in `CLAUDE.md`.
> For WAME packages it is `Wame`; the full package namespace is
> `{{NAMESPACE}}\{{CLASS_PREFIX}}` → `Wame\ExampleWidget`.

## Directory layout

```
wamesk/example-widget/
├── composer.json                     ← stubs/composer.json.stub
├── config/
│   └── example-widget.php            (optional, published via vendor:publish)
├── database/
│   ├── factories/                    (tests only — PSR-4 mapped)
│   ├── migrations/
│   │   └── <ts>_create_example_widgets_table.php  ← stubs/migration.php.stub
│   └── seeders/                      (tooling only — DO NOT ship data here)
├── lang/
│   └── en/
│       └── messages.php              (optional, loadTranslationsFrom)
├── resources/
│   └── views/
│       └── example.blade.php         ← stubs/view.blade.php.stub
├── routes/
│   └── web.php                       ← stubs/web-routes.php.stub
├── src/
│   ├── ExampleWidgetServiceProvider.php   ← stubs/ServiceProvider.php.stub
│   ├── Http/Controllers/
│   ├── Models/
│   └── Services/
└── tests/
    └── ExampleWidgetServiceProviderTest.php  ← stubs/ServiceProviderTest.php.stub
```

## What each stub becomes

| Stub                               | Destination                                              | Purpose                                           |
|------------------------------------|---------------------------------------------------------|---------------------------------------------------|
| `composer.json.stub`               | `composer.json`                                          | Package metadata, PSR-4, provider auto-discovery  |
| `ServiceProvider.php.stub`         | `src/{{CLASS_PREFIX}}ServiceProvider.php`               | register()/boot(), publishing, resource loading   |
| `ServiceProviderTest.php.stub`     | `tests/{{CLASS_PREFIX}}ServiceProviderTest.php`         | Pest test asserting the provider boots            |
| `migration.php.stub`               | `database/migrations/<ts>_create_{{TABLE}}_table.php`  | Anonymous-class migration, `dateTimeTz()` columns |
| `web-routes.php.stub`              | `routes/web.php`                                         | RESTful route skeleton (commented)                |
| `view.blade.php.stub`             | `resources/views/example.blade.php`                     | Minimal Blade view                                |

## PSR-4 mapping (from composer.json.stub)

All four namespaces map onto `{{NAMESPACE}}\{{CLASS_PREFIX}}`:

- `{{NAMESPACE}}\{{CLASS_PREFIX}}\` → `src/`
- `{{NAMESPACE}}\{{CLASS_PREFIX}}\Tests\` → `tests/`
- `{{NAMESPACE}}\{{CLASS_PREFIX}}\Database\Factories\` → `database/factories/`
- `{{NAMESPACE}}\{{CLASS_PREFIX}}\Database\Seeders\` → `database/seeders/`

The ServiceProvider lives at the **namespace root** (`src/{{CLASS_PREFIX}}ServiceProvider.php`),
so `extra.laravel.providers` points at `{{NAMESPACE}}\{{CLASS_PREFIX}}\{{CLASS_PREFIX}}ServiceProvider`.

## Data & defaults — migrations, not seeders

- **Reference/lookup data and defaults** ship via idempotent `*_seed_*` migrations
  whose `up()` writes rows only when absent (check-then-insert). A plain
  `php artisan migrate` reproduces full state — no seeder run required.
- **Factories** stay under `database/factories/` for tests only.
- The `database/seeders/` PSR-4 mapping exists only for tooling compatibility.
  Do not ship data there.
- Static/lookup data may alternatively be modelled with **Laravel Sushi** (array-backed
  Eloquent models) instead of a table + seed migration.

## Datetime standard

Migrations use `dateTimeTz()` for `created_at` / `updated_at` / `deleted_at`.
Never `timestamps()`, `softDeletes()`, or plain `datetime()`. The migration stub
already follows this — keep it when filling in columns.
