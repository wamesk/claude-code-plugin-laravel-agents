# Testing Patterns (Pest, non-Nova)

Feature tests for APIs, services, actions, auth, and database state. Always Pest syntax — never PHPUnit class syntax. Browser/Nova (Dusk) tests are out of scope for this skill. Placeholders `Vendor\Module` / `vendor/module` map to the real names in `CLAUDE.md`.

## Pest syntax (mandatory)

```php
<?php

declare(strict_types = 1);

use Vendor\Module\Models\User;

test('user can be created via API', function () {
    $response = $this->postJson('/api/v1/users', [
        'name' => 'John Doe',
        'email' => 'john@example.com',
        'password' => 'password123',
        'password_confirmation' => 'password123',
    ]);

    $response->assertStatus(201)
        ->assertJson([
            'type' => 'success',
            'code' => 'module::user.store.success',
        ])
        ->assertJsonStructure([
            'type',
            'code',
            'data' => ['id', 'name', 'email'],
        ]);

    $this->assertDatabaseHas('users', [
        'email' => 'john@example.com',
    ]);
});
```

```php
// ❌ WRONG — never PHPUnit class syntax
class UserControllerTest extends TestCase
{
    public function test_user_can_be_created() { /* ... */ }
}
```

## API response testing

Assert the envelope (`type`/`code`), the payload structure, and the resulting database state. Cover success **and** error paths.

```php
test('index returns the standard success envelope', function () {
    User::factory()->count(5)->create();

    $this->getJson('/api/v1/users')
        ->assertStatus(200)
        ->assertJson([
            'type' => 'success',
            'code' => 'module::user.index.success',
        ])
        ->assertJsonStructure([
            'type',
            'code',
            'data' => [
                '*' => ['id', 'name', 'email', 'created_at', 'updated_at'],
            ],
        ]);
});

test('store validation returns the error envelope', function () {
    $this->postJson('/api/v1/users', [
        'name' => '',              // invalid
        'email' => 'invalid-email', // invalid
    ])
        ->assertStatus(400)
        ->assertJson([
            'type' => 'error',
            'code' => 'module::user.store.error',
        ])
        ->assertJsonStructure(['type', 'code', 'message']);
});

test('show returns 404 for a missing record', function () {
    $this->getJson('/api/v1/users/non-existent-id')
        ->assertStatus(404)
        ->assertJson([
            'type' => 'error',
            'code' => 'module::user.show.error',
        ]);
});
```

## Testing services

```php
<?php

declare(strict_types = 1);

use Vendor\Module\Models\User;
use Vendor\Module\Services\UserService;

beforeEach(function () {
    $this->userService = app(UserService::class);
});

test('service creates a user', function () {
    $user = $this->userService->create(
        name: 'John Doe',
        email: 'john@example.com',
        password: 'password123',
    );

    expect($user)->toBeInstanceOf(User::class)
        ->and($user->email)->toBe('john@example.com');

    $this->assertDatabaseHas('users', ['email' => 'john@example.com']);
});

test('service paginates users', function () {
    User::factory()->count(25)->create();

    $paginated = $this->userService->getAllPaginated(perPage: 10);

    expect($paginated)->toHaveCount(10)
        ->and($paginated->total())->toBe(25)
        ->and($paginated->lastPage())->toBe(3);
});
```

## Testing actions

```php
<?php

declare(strict_types = 1);

use Vendor\Module\Actions\CreateUserAction;
use Vendor\Module\Models\User;

test('action creates a user with a hashed password', function () {
    $user = (new CreateUserAction())->handle([
        'name' => 'John Doe',
        'email' => 'john@example.com',
        'password' => 'password123',
    ]);

    expect($user)->toBeInstanceOf(User::class)
        ->and(password_verify('password123', $user->password))->toBeTrue();
});
```

## Authentication & authorization (Sanctum)

```php
<?php

declare(strict_types = 1);

use Laravel\Sanctum\Sanctum;
use Vendor\Module\Models\User;

test('guests cannot access protected routes', function () {
    $this->getJson('/api/v1/users')->assertStatus(401);
});

test('authenticated users can access protected routes', function () {
    Sanctum::actingAs(User::factory()->create());

    $this->getJson('/api/v1/users')->assertStatus(200);
});

test('a user cannot update another user', function () {
    Sanctum::actingAs(User::factory()->create());
    $other = User::factory()->create();

    $this->putJson("/api/v1/users/{$other->id}", ['name' => 'X'])
        ->assertStatus(403);
});
```

## Database state

```php
test('a record can be soft deleted', function () {
    $user = User::factory()->create();

    $user->delete();

    $this->assertSoftDeleted('users', ['id' => $user->id]);
});

test('eager loading prevents N+1', function () {
    User::factory()->has(Profile::factory())->count(10)->create();

    DB::enableQueryLog();

    User::with('profile')->get()->each(fn ($u) => $u->profile->name);

    // 1 query for users + 1 for profiles.
    expect(DB::getQueryLog())->toHaveCount(2);
});
```

## Test helpers (traits)

Extract repeated assertions into reusable traits.

```php
<?php

declare(strict_types = 1);

namespace Tests\Helpers;

use Vendor\Module\Models\User;

trait ApiTestHelper
{
    protected function assertApiSuccessResponse($response, string $code, int $status = 200): void
    {
        $response->assertStatus($status)
            ->assertJson(['type' => 'success', 'code' => $code])
            ->assertJsonStructure(['type', 'code', 'data']);
    }

    protected function assertApiErrorResponse($response, string $code, int $status = 400): void
    {
        $response->assertStatus($status)
            ->assertJson(['type' => 'error', 'code' => $code])
            ->assertJsonStructure(['type', 'code', 'message']);
    }

    protected function authenticateUser(?User $user = null): array
    {
        $user = $user ?? User::factory()->create();
        $token = $user->createToken('test-token')->plainTextToken;

        return [
            'user' => $user,
            'token' => $token,
            'headers' => [
                'Authorization' => "Bearer {$token}",
                'Accept' => 'application/json',
            ],
        ];
    }
}
```

## Factories

Factories exist **for tests only** (and local exploration) — never as the way to ship production data (that is a seed migration; see `database-conventions.md`). Register with `#[UseFactory]` (see `service-layer.md`).

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Vendor\Module\Models\User;

class UserFactory extends Factory
{
    protected $model = User::class;

    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => Hash::make('password'),
            'remember_token' => Str::random(10),
        ];
    }

    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }
}
```

## Scenario builders

Encapsulate common object graphs behind named factories.

```php
<?php

declare(strict_types = 1);

namespace Tests\Helpers;

use Vendor\Module\Models\User;

class UserTestScenario
{
    public static function userWithContent(): User
    {
        return User::factory()
            ->has(Profile::factory())
            ->has(Post::factory()->count(5))
            ->create();
    }

    public static function adminUser(): User
    {
        return User::factory()->create(['role' => 'admin', 'status' => 'active']);
    }
}
```

## Datasets

Drive one test with many inputs.

```php
test('email validation rejects invalid formats', function (string $email) {
    $this->postJson('/api/v1/users', [
        'name' => 'John Doe',
        'email' => $email,
        'password' => 'password123',
    ])
        ->assertStatus(400)
        ->assertJsonValidationErrors('email');
})->with([
    'missing @' => 'invalidemail.com',
    'missing domain' => 'invalid@',
    'spaces' => 'invalid @email.com',
]);
```

## Naming & organization

```php
// ✅ descriptive, action-based
test('user cannot create an account with a duplicate email');
test('guest cannot access protected API endpoints');

// ❌ vague / technical
test('test_create_user');
test('it works');
```

- One assertion focus per test; arrange-act-assert.
- Isolated and deterministic — no interdependencies, mock external services.
- Place each test in the module it belongs to (`vendor/module/tests/Feature/...`).

## Translation-key pattern

No hardcoded user-facing strings — everything is a translation key, in Blade/Vue views, Filament/Nova fields, notifications, emails, and API responses alike. The **key** is always English words (`field.name.help`, never a translated phrase); the **value** in the language file is the target language. Translations are **modular** — one file per module/entity, never one shared catch-all file. Which languages a project ships, and whether translation into all of them is expected automatically, is defined in that project's own `CLAUDE.md` — check it before assuming a list. Response `code` values, validation messages, and field labels all use the same structure:

```
module::entity.field.label
module::entity.field.help
module::entity.field.<rule>     // e.g. .required
module::entity.action.success   // e.g. store.success
module::entity.action.error
```

Usage:

```php
// Controller — code (key) + translated message.
'code' => 'module::user.store.success',
'message' => __('module::user.store.success'),

// FormRequest — attribute/message keys.
'name.required' => __('module::user.field.name.required'),
```

Translation file (`resources/lang/en/user.php` inside the module):

```php
return [
    'plural' => 'Users',
    'singular' => 'User',

    'field.name.label' => 'Name',
    'field.name.help' => 'The user\'s full name',
    'field.email.label' => 'Email',

    'store.success' => 'User created successfully',
    'store.error' => 'Failed to create the user',
];
```

Tests assert on the `code` key (stable), not the translated sentence — as shown in the API examples above.

## Checklist

- ✅ Pest syntax only; descriptive, action-based test names.
- ✅ Assert the `type`/`code` envelope, the JSON structure, and the database state.
- ✅ Cover both success and error scenarios; use datasets for input matrices.
- ✅ Factories for test data; scenario builders for common graphs; helper traits for repeated assertions.
- ✅ Factories are for tests only — never for shipping data.
- ✅ Isolated, deterministic tests placed in the owning module.
- ✅ Assert on translation `code` keys, not translated strings.
