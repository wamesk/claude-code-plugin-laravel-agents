# Database Conventions

Schema and migration standards. Placeholders `Vendor\Module` / `vendor/module` map to the real namespace/package defined in `CLAUDE.md`.

## ULID primary keys (default)

```php
Schema::create('posts', function (Blueprint $table): void {
    $table->ulid('id')->primary();
    $table->string('title', 200);
    $table->string('slug', 220)->unique();
    $table->dateTimeTz('created_at');
    $table->dateTimeTz('updated_at');
    $table->dateTimeTz('deleted_at')->nullable();
});
```

The model opts in with `HasUlids`:

```php
use Illuminate\Database\Eloquent\Concerns\HasUlids;

class Post extends Model
{
    use HasUlids;
}
```

### Documented exceptions to ULID

Some tables intentionally use string or big-integer keys:

```php
// ISO 3166-1 alpha-2 country codes.
Schema::create('countries', function (Blueprint $table): void {
    $table->char('id', 2)->primary();
    $table->char('language_id', 5)->nullable();
    $table->char('currency_id', 3)->nullable();
    $table->string('title', 40);
    $table->unsignedTinyInteger('sort')->default(0)->index();
    $table->boolean('status')->default(CountryStatusEnum::ENABLED)->index();
    $table->dateTimeTz('created_at');
    $table->dateTimeTz('updated_at');
    $table->dateTimeTz('deleted_at')->nullable();

    $table->index(['status', 'id']);
});

// Third-party/framework packages that mandate an auto-increment big integer.
Schema::create('notifications', function (Blueprint $table): void {
    $table->id(); // bigInteger autoincrement — required by the package
    $table->string('type');
    $table->morphs('notifiable');
    $table->text('data');
    $table->dateTimeTz('read_at')->nullable();
    $table->dateTimeTz('created_at')->nullable();

    $table->index(['notifiable_type', 'notifiable_id']);
});
```

For purely static reference codes (e.g. ISO currencies) prefer Laravel Sushi over a table — see the bottom of this file.

## Foreign keys with cascade rules

```php
// Optional relationship — null the child when the parent is deleted.
$table->foreignUlid('author_id')
    ->nullable()
    ->constrained('users')
    ->cascadeOnUpdate()
    ->nullOnDelete();

// Required relationship — delete the child with the parent.
$table->foreignUlid('post_id')
    ->constrained('posts')
    ->cascadeOnUpdate()
    ->cascadeOnDelete();

// String-keyed parent — prevent deletion while children exist.
$table->char('country_id', 2);
$table->foreign('country_id')
    ->references('id')->on('countries')
    ->cascadeOnUpdate()
    ->restrictOnDelete();
```

Always add an index for each foreign key column (see below).

## Indexes (simple & composite)

```php
// Simple indexes on frequently queried columns.
$table->index('slug');
$table->index('status');
$table->index('created_at');

// Composite indexes — column order matters, most selective first,
// matched to real query patterns.
// Query: WHERE author_id = ? AND status = ? ORDER BY created_at DESC
$table->index(['author_id', 'status', 'created_at']);
// Query: WHERE status = ? AND created_at > ?
$table->index(['status', 'created_at']);
```

Full-text indexes are added in their own migration:

```php
Schema::table('posts', function (Blueprint $table): void {
    $table->fullText(['title', 'content']);
});
```

## Column-type optimization

Use the smallest type that fits the domain, and `unsigned` for non-negative values.

```php
$table->unsignedTinyInteger('status')->default(1);       // 0–255
$table->unsignedSmallInteger('stock')->default(0);       // 0–65 535
$table->unsignedMediumInteger('total_sales')->default(0);// 0–16 777 215
$table->unsignedInteger('price_cents');                  // 0–~4B (money in cents)
$table->unsignedBigInteger('views_all_time')->default(0);

$table->string('email', 254)->unique();  // RFC 5321 max — NOT 255
$table->string('name', 100);             // size to the domain — NOT 255
$table->string('phone', 20)->nullable();

$table->text('bio')->nullable();          // < 64 KB
$table->mediumText('description')->nullable(); // < 16 MB

$table->json('specifications')->nullable(); // cast to 'array' on the model
```

## Enums for boolean & static values

Never use `boolean()`; model both booleans and small fixed sets as backed PHP enums stored in a `tinyInteger`/short `string`.

```php
// ❌ WRONG
$table->boolean('is_active')->default(true);

// ✅ CORRECT
$table->unsignedTinyInteger('status')->default(PostStatusEnum::ENABLED);
```

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Enums;

enum PostStatusEnum: int
{
    case DISABLED = 0;
    case ENABLED = 1;

    public function label(): string
    {
        return match ($this) {
            self::DISABLED => __('module::post.status.disabled'),
            self::ENABLED => __('module::post.status.enabled'),
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::DISABLED => 'gray',
            self::ENABLED => 'green',
        };
    }

    public function isEnabled(): bool
    {
        return $this === self::ENABLED;
    }
}
```

String-backed enum for static values:

```php
enum PostVisibilityEnum: string
{
    case PRIVATE = 'private';
    case PUBLIC = 'public';
}
```

Cast on the model — note the migration column is `dateTimeTz`, while the Eloquent **cast** for a datetime remains `'datetime'`:

```php
protected function casts(): array
{
    return [
        'status' => PostStatusEnum::class,
        'visibility' => PostVisibilityEnum::class,
        'specifications' => 'array',
        'published_at' => 'datetime',
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
    ];
}
```

## `sort` column, not `order`

`order` is a reserved word in MySQL — always name ordering columns `sort`.

```php
// ✅ CORRECT
$table->unsignedTinyInteger('sort')->default(0)->index();

// ❌ WRONG
$table->unsignedTinyInteger('order')->default(0);
```

## `dateTimeTz` for all datetimes

Every datetime column — including `created_at`, `updated_at`, and `deleted_at` — uses `dateTimeTz()` for timezone-aware storage. Never use `timestamps()`, `timestamp()`, `datetime()`, `softDeletes()`, or `softDeletesDatetime()`.

```php
Schema::create('events', function (Blueprint $table): void {
    $table->ulid('id')->primary();
    $table->string('name', 200);

    $table->dateTimeTz('start_date');
    $table->dateTimeTz('end_date')->nullable();
    $table->dateTimeTz('published_at')->nullable();

    // Timestamps, written explicitly with timezone.
    $table->dateTimeTz('created_at');
    $table->dateTimeTz('updated_at');
    $table->dateTimeTz('deleted_at')->nullable(); // soft-delete column

    $table->index(['start_date', 'end_date']);
});
```

Soft deletes still use the `SoftDeletes` trait on the model — only the migration column changes to `dateTimeTz('deleted_at')->nullable()`.

## Migration best practices

- **Never modify a shipped migration.** Add a new one for every schema change.

```php
// 2024_01_15_000000_add_phone_to_users_table.php
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table): void {
            $table->string('phone', 20)->nullable()->after('email');
            $table->index('phone');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table): void {
            $table->dropIndex(['phone']);
            $table->dropColumn('phone');
        });
    }
};
```

- Provide a real, reversible `down()`.
- Populate/backfill data in chunks inside the same migration when adding a non-nullable column derived from existing rows.

## Ship data via seed migrations, NOT seeders

**Do not use database seeders to ship data or defaults.** The project must reproduce its full state with `php artisan migrate` alone — no seeder step. Any default value, reference/lookup row, or system default ships as an **idempotent `*_seed_*.php` migration** whose `up()` writes a row only when its key is absent.

- Default to a seed migration unless explicitly told otherwise, so a plain `migrate` on a fresh or restored database applies everything.
- If something looks like it wants a seeder (defaults, reference data, demo rows), **stop and ask** whether it should be a migration or a seeder before implementing.

```php
<?php

declare(strict_types = 1);

use Illuminate\Database\Migrations\Migration;
use Vendor\Module\Models\Category;

return new class extends Migration
{
    /**
     * Seed the default categories.
     *
     * Idempotent: firstOrCreate only inserts when the unique key is
     * absent, so re-running migrate (fresh or restored DB) reproduces
     * the full state without ever running a seeder.
     */
    public function up(): void
    {
        $defaults = [
            ['slug' => 'news', 'title' => 'News'],
            ['slug' => 'blog', 'title' => 'Blog'],
        ];

        foreach ($defaults as $row) {
            Category::query()->firstOrCreate(
                ['slug' => $row['slug']],
                $row,
            );
        }
    }

    public function down(): void
    {
        Category::query()
            ->whereIn('slug', ['news', 'blog'])
            ->delete();
    }
};
```

Name such migrations `..._seed_<what>.php` (e.g. `2024_01_10_000000_seed_default_categories.php`). The same pattern applies to any default (system settings, lookup rows, etc.): write it as an idempotent seed migration that only sets a value when the key is missing.

**Factories still exist** — but strictly for tests (and local exploration), never as the mechanism for shipping production data. See `testing-patterns.md`.

## Static data with Laravel Sushi

For small, immutable reference data (currencies, fixed lists), use [Laravel Sushi](https://usesushi.dev/) instead of a table + seed migration:

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Models;

use Illuminate\Database\Eloquent\Model;
use Sushi\Sushi;

class Currency extends Model
{
    use Sushi;

    protected $primaryKey = 'id';
    protected $keyType = 'string';
    public $incrementing = false;

    protected $rows = [
        ['id' => 'EUR', 'name' => 'Euro', 'symbol' => '€', 'decimal_places' => 2],
        ['id' => 'USD', 'name' => 'US Dollar', 'symbol' => '$', 'decimal_places' => 2],
        ['id' => 'CZK', 'name' => 'Czech Koruna', 'symbol' => 'Kč', 'decimal_places' => 2],
    ];
}
```

Sushi models behave like ordinary Eloquent models (`Currency::all()`, `Currency::find('EUR')`, relationships) but are backed by the in-memory `$rows` array.

## Checklist

- ✅ ULID primary keys by default (string/big-int only for documented exceptions).
- ✅ Enums for boolean and static values — never `boolean()`.
- ✅ Smallest appropriate, `unsigned` integer types; string lengths sized to the domain (not 255).
- ✅ `dateTimeTz()` for every datetime, including `created_at`/`updated_at`/`deleted_at`.
- ✅ Never `timestamps()`, `softDeletes()`, `timestamp()`, or plain `datetime()`.
- ✅ `sort`, never `order`.
- ✅ `foreignUlid()`/`foreign()` with explicit cascade rules; index every FK.
- ✅ Composite indexes matched to real query patterns.
- ✅ New migration per change; never edit a shipped one; reversible `down()`.
- ✅ Data/defaults ship via idempotent `*_seed_*.php` migrations, never seeders.
- ✅ Small immutable reference data via Laravel Sushi.
