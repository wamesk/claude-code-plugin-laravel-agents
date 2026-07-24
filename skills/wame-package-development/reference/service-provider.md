# Service provider

The service provider is the package's entry point. Follows
https://www.laravelpackage.com. See `stubs/ServiceProvider.php.stub`.

## register() vs boot()

| Method       | Runs                                          | Do                                                                 | Don't                                                            |
|--------------|-----------------------------------------------|-------------------------------------------------------------------|-----------------------------------------------------------------|
| `register()` | For every request, **before** all providers boot | Bind services (`bind`, `singleton`), `mergeConfigFrom()`          | Resolve other services, hit the DB, load routes/views, publish  |
| `boot()`     | **After** all providers are registered        | Load routes/views/translations/migrations, register publishes, events | Assume request-specific state that isn't ready yet              |

Rule of thumb: **wiring** goes in `register()`, **loading & exposing resources**
goes in `boot()`. Nothing in `register()` may depend on another provider having
booted.

## Merging config (register)

`mergeConfigFrom()` gives the host sane defaults even when the config has not been
published, and lets the published file override individual keys:

```php
public function register(): void
{
    $this->mergeConfigFrom(__DIR__ . '/../config/example-widget.php', 'example-widget');
}
```

## Loading resources (boot)

```php
public function boot(): void
{
    $this->loadMigrationsFrom(__DIR__ . '/../database/migrations');
    $this->loadRoutesFrom(__DIR__ . '/../routes/web.php');
    $this->loadViewsFrom(__DIR__ . '/../resources/views', 'example-widget');
    $this->loadTranslationsFrom(__DIR__ . '/../lang', 'example-widget');
}
```

- Views become `view('example-widget::example')`.
- Translations become `__('example-widget::messages.title')`.
- `loadMigrationsFrom()` runs the package migrations directly on
  `php artisan migrate` — the host does **not** need to publish them first.

## Publishing assets (console only)

Wrap publishes in `runningInConsole()` and give each group a tag so the host can
copy assets selectively:

```php
if ($this->app->runningInConsole()) {
    $this->publishes([
        __DIR__ . '/../config/example-widget.php' => config_path('example-widget.php'),
    ], 'example-widget-config');

    $this->publishes([
        __DIR__ . '/../database/migrations' => database_path('migrations'),
    ], 'example-widget-migrations');

    $this->publishes([
        __DIR__ . '/../resources/views' => resource_path('views/vendor/example-widget'),
    ], 'example-widget-views');
}
```

Host commands:

```bash
php artisan vendor:publish --tag=example-widget-config
php artisan vendor:publish --tag=example-widget-migrations
php artisan migrate
```

**loadMigrationsFrom vs publishing migrations:** prefer `loadMigrationsFrom()` so
migrations run from the package and stay versioned with it. Only publish migrations
when the host must edit the schema. Either way, **shipped data/defaults go in
idempotent `*_seed_*` migrations, never seeders** (factories are for tests only).

## Registering commands, gates, etc. (boot)

```php
if ($this->app->runningInConsole()) {
    $this->commands([
        \Wame\ExampleWidget\Console\SyncCommand::class,
    ]);
}
```

## Supporting new Laravel versions

Per laravelpackage.com "Supporting New Laravel Versions":

- Keep `illuminate/*` (or `laravel/framework`) constraints **wide** — use OR ranges
  so one release supports several majors. Use the supported Laravel/Pest version
  from the **Supported versions** table in `SKILL.md`; do not pin a single major.
- Prefer `illuminate/support` (and other granular `illuminate/*` packages) over
  requiring the whole `laravel/framework` in a library.
- Test against each supported major in CI (a matrix) before widening the range.
- Bump the dev tooling (`pestphp/pest`, `orchestra/testbench`) in step with the
  Laravel majors you claim to support — Testbench versions track framework majors.
- When adding support for a new major, widen the constraint, run the suite on that
  major, then tag a release; drop an old major only in a new major of your package.
