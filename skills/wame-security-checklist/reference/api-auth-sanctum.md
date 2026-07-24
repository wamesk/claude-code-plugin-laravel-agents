# API Authentication (Sanctum), Authorization & Testing

Sanctum token lifecycle, login hardening, session security, authorization and
ownership checks, and the security testing commands.

---

## Sanctum Token Security

```php
// config/sanctum.php
return [
    // Tokens expire (in minutes); null = never (avoid in production)
    'expiration' => 60 * 24, // 24 hours

    'middleware' => [
        'encrypt_cookies'    => App\Http\Middleware\EncryptCookies::class,
        'validate_csrf_token' => App\Http\Middleware\VerifyCsrfToken::class,
    ],
];
```

```php
// CORRECT - Issue a token with an explicit expiration and least-privilege abilities
public function login(LoginRequest $request): JsonResponse
{
    if (! Auth::attempt($request->only('email', 'password'))) {
        return response()->json([
            'type'    => 'error',
            'code'    => 'auth::invalid_credentials',
            'message' => __('auth::invalid_credentials'),
        ], Response::HTTP_UNAUTHORIZED);
    }

    $user = Auth::user();

    $token = $user->createToken(
        'auth-token',
        ['*'],              // Scope abilities down per client where possible
        now()->addDay(),    // Expiration
    )->plainTextToken;

    return response()->json([
        'type' => 'success',
        'code' => 'auth::login.success',
        'data' => [
            'token'      => $token,
            'expires_at' => now()->addDay()->toIso8601String(),
            'user'       => UserResource::make($user),
        ],
    ], Response::HTTP_OK);
}
```

```php
// CORRECT - Logout revokes the current token
public function logout(Request $request): JsonResponse
{
    $request->user()->currentAccessToken()->delete();

    return response()->json([
        'type'    => 'success',
        'code'    => 'auth::logout.success',
        'message' => __('auth::logout.success'),
    ], Response::HTTP_OK);
}

// CORRECT - Logout everywhere revokes all tokens (e.g. after a password change)
public function logoutAll(Request $request): JsonResponse
{
    $request->user()->tokens()->delete();

    return response()->json([
        'type'    => 'success',
        'code'    => 'auth::logout_all.success',
        'message' => __('auth::logout_all.success'),
    ], Response::HTTP_OK);
}
```

Rules:
- Always set a token expiration; revoke tokens on logout and password change.
- Never store the plain-text token server-side; return it once at creation.
- Scope token abilities to what the client needs instead of `['*']`.

---

## Login Hardening (rate limit + lockout)

```php
// CORRECT - Throttle at the route AND track failed attempts per account
Route::middleware(['throttle:5,1'])->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
});

use Illuminate\Support\Facades\RateLimiter;

public function login(LoginRequest $request): JsonResponse
{
    $email = $request->validated('email');
    $key   = 'login:'.$email;

    // Locked out after too many failures
    if (RateLimiter::tooManyAttempts($key, 5)) {
        $seconds = RateLimiter::availableIn($key);

        return response()->json([
            'type'    => 'error',
            'code'    => 'auth::too_many_attempts',
            'message' => __('auth::too_many_attempts', ['seconds' => $seconds]),
        ], Response::HTTP_TOO_MANY_REQUESTS);
    }

    if (Auth::attempt($request->only('email', 'password'))) {
        RateLimiter::clear($key); // Reset on success

        $user  = Auth::user();
        $token = $user->createToken('auth-token', ['*'], now()->addDay())->plainTextToken;

        return response()->json([
            'type' => 'success',
            'code' => 'auth::login.success',
            'data' => ['token' => $token, 'user' => UserResource::make($user)],
        ], Response::HTTP_OK);
    }

    RateLimiter::hit($key, 300); // Count this failure; window 5 minutes

    return response()->json([
        'type'    => 'error',
        'code'    => 'auth::invalid_credentials',
        'message' => __('auth::invalid_credentials'),
    ], Response::HTTP_UNAUTHORIZED);
}
```

Return the same generic message and status for "unknown email" and "wrong
password" so the endpoint does not disclose which accounts exist.

---

## Session Security (stateful / SPA cookie auth)

```php
// config/session.php
return [
    'driver'         => env('SESSION_DRIVER', 'redis'), // redis in production
    'lifetime'       => 120,
    'expire_on_close' => true,

    'encrypt'   => true,   // Encrypt session payloads
    'http_only' => true,   // No JavaScript access to the cookie
    'same_site' => 'strict', // CSRF hardening
    'secure'    => env('SESSION_SECURE_COOKIE', true), // HTTPS only in prod
];
```

---

## Authorization & Ownership Checks

Authentication proves who the caller is; authorization proves they may act on
this specific record. Missing the second step is the root cause of IDOR (A01).

```php
// Policy - the single source of truth for who may do what
class PostPolicy
{
    public function view(User $user, Post $post): bool
    {
        return $post->user_id === $user->id || $user->isAdmin();
    }

    public function update(User $user, Post $post): bool
    {
        return $post->user_id === $user->id || $user->isAdmin();
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->isAdmin();
    }
}
```

```php
// Controller - authorize on EVERY record-scoped action
public function update(UpdatePostRequest $request, Post $post): JsonResponse
{
    $this->authorize('update', $post); // 403 if not the owner/admin

    $post->update($request->validated());

    return response()->json([
        'type' => 'success',
        'code' => 'post::post.update.success',
        'data' => PostResource::make($post),
    ], Response::HTTP_OK);
}
```

Patterns that prevent IDOR:
- Resolve nested records through the parent relation so ownership is implicit,
  e.g. `$request->user()->posts()->findOrFail($id)` instead of `Post::find($id)`.
- Call `$this->authorize(...)` (or a Gate) before returning, updating or deleting
  any record identified by a client-supplied id.
- Never trust a `user_id`/`owner_id` sent in the request body; derive it from the
  authenticated user.

---

## Security Testing Commands

```bash
# Report dependencies with known vulnerabilities
composer audit

# List outdated packages
composer outdated

# Static analysis at the strictest level (if installed)
./vendor/bin/phpstan analyse --level=8

# Verify .env permissions (should be 600)
ls -la .env

# Probe rate limiting on the login endpoint
for i in {1..10}; do curl -X POST https://api.example.com/login; done

# Check for accidentally exposed files (each should 403/404, never 200)
curl -I https://example.com/.env
curl -I https://example.com/.git/config
curl -I https://example.com/composer.json
```

Verify authorization with automated tests: assert that user B receives 403/404
when requesting user A's record, and that expired/revoked tokens are rejected.
