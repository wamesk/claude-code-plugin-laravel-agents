# OWASP Top 10 (2021) — Laravel Security Checks

Each item lists what to look for and shows vulnerable versus secure code. For
authentication/authorization mechanics (Sanctum, sessions, ownership checks) see
`api-auth-sanctum.md`. For mass assignment, file uploads, CSRF/XSS, rate
limiting, CORS and secrets/env see `laravel-security.md`.

> The `Vendor\Module` namespace used in the middleware examples below is a
> placeholder — the real namespace is defined per-project in `CLAUDE.md`.

---

## A01:2021 — Broken Access Control

### What to check
- Missing authentication middleware on routes.
- Inadequate authorization checks (no policy/gate).
- Insecure Direct Object Reference (IDOR).
- Missing policy checks on show/update/delete.
- Privilege escalation.
- Path traversal.

### IDOR and authorization

```php
// CRITICAL - No authentication check at all
Route::get('/users/{id}', [UserController::class, 'show']);

// CRITICAL - Authenticated, but any user can read any user's data (IDOR)
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/users/{id}', [UserController::class, 'show']);
});

public function show(string $id): JsonResponse
{
    // No authorization check - user can access any user's data!
    $user = User::findOrFail($id);

    return response()->json(['data' => UserResource::make($user)]);
}
```

```php
// CORRECT - Authorization via policy
public function show(string $id): JsonResponse
{
    try {
        $user = User::findOrFail($id);

        $this->authorize('view', $user);

        return response()->json([
            'type' => 'success',
            'code' => 'user::user.show.success',
            'data' => UserResource::make($user),
        ], Response::HTTP_OK);
    } catch (AuthorizationException $e) {
        return response()->json([
            'type' => 'error',
            'code' => 'user::user.unauthorized',
            'message' => __('user::user.unauthorized'),
        ], Response::HTTP_FORBIDDEN);
    }
}
```

```php
// UserPolicy.php
class UserPolicy
{
    public function view(User $currentUser, User $user): bool
    {
        // Own profile, or an admin
        return $currentUser->id === $user->id || $currentUser->isAdmin();
    }

    public function update(User $currentUser, User $user): bool
    {
        return $currentUser->id === $user->id || $currentUser->isAdmin();
    }

    public function delete(User $currentUser, User $user): bool
    {
        // Only admins can delete users
        return $currentUser->isAdmin();
    }
}
```

### Path traversal protection

```php
// CRITICAL - Path traversal (attacker sends ../../.env)
public function download(Request $request): BinaryFileResponse
{
    $filename = $request->input('file');
    $path = storage_path("documents/{$filename}");

    return response()->download($path);
}
```

```php
// CORRECT - Validate the filename and confine it to the allowed directory
public function download(Request $request): BinaryFileResponse
{
    $filename = $request->validated('file');

    if (! preg_match('/^[a-zA-Z0-9_\-.]+$/', $filename)) {
        abort(400, 'Invalid filename');
    }

    $path = storage_path("documents/{$filename}");

    $realPath = realpath($path);
    $allowedDir = realpath(storage_path('documents'));

    if (! $realPath || ! str_starts_with($realPath, $allowedDir)) {
        abort(404, 'File not found');
    }

    if (! file_exists($realPath)) {
        abort(404, 'File not found');
    }

    return response()->download($realPath);
}
```

---

## A02:2021 — Cryptographic Failures

### What to check
- Passwords stored in plain text.
- Weak hashing (MD5/SHA1).
- Hardcoded secrets.
- Unencrypted sensitive data at rest.
- No HTTPS / insecure transport.
- Weak encryption algorithms.

```php
// CRITICAL - Plain text password
User::create([
    'password' => $request->input('password'), // NEVER
]);

// CRITICAL - Weak hashing
User::create([
    'password' => md5($request->input('password')), // INSECURE
]);

// CRITICAL - Hardcoded secret
$apiKey = 'sk_live_123456789abcdef'; // NEVER hardcode secrets
```

```php
// CORRECT - Hash passwords with bcrypt/argon2id
use Illuminate\Support\Facades\Hash;

User::create([
    'name'     => $request->validated('name'),
    'email'    => $request->validated('email'),
    'password' => Hash::make($request->validated('password')),
]);

// CORRECT - Read secrets from config (backed by env)
$apiKey = config('services.payment.secret');

// config/services.php
return [
    'payment' => [
        'secret' => env('PAYMENT_SECRET_KEY'),
    ],
];

// CORRECT - Encrypt sensitive columns at rest
class User extends Model
{
    protected $casts = [
        'ssn'         => 'encrypted',
        'credit_card' => 'encrypted',
        'phone'       => 'encrypted',
    ];
}

// CORRECT - Constant-time password verification
if (Hash::check($plainPassword, $hashedPassword)) {
    // Password is correct
}
```

Enforce HTTPS in production via `.env` (`APP_URL=https://...`,
`SESSION_SECURE_COOKIE=true`) — see `laravel-security.md`.

### Hashing configuration

```php
// config/hashing.php
return [
    'driver' => 'bcrypt',

    'bcrypt' => [
        'rounds' => env('BCRYPT_ROUNDS', 12), // Minimum 10, recommended 12+
    ],

    // Or argon2id (more resistant to GPU attacks)
    'argon' => [
        'memory'  => 65536,
        'threads' => 4,
        'time'    => 4,
    ],
];
```

---

## A03:2021 — Injection

### What to check
- SQL injection (string concatenation, `whereRaw`, raw `DB::select`).
- Command injection (`exec`, `shell_exec`, `system` with user input).
- Unvalidated user input reaching any interpreter.

### SQL injection

```php
// CRITICAL - String concatenation into SQL
$search = $request->input('query');
$users = DB::select("SELECT * FROM users WHERE name LIKE '%{$search}%'");

// CRITICAL - whereRaw with interpolation is still injectable
$users = User::whereRaw("name LIKE '%{$search}%'")->get();
```

```php
// CORRECT - Eloquent with bound parameters
$search = $request->validated('query');

$users = User::query()
    ->where('name', 'like', "%{$search}%")
    ->get();

// CORRECT - Query Builder with bindings
$users = DB::table('users')
    ->where('name', 'like', "%{$search}%")
    ->get();

// CORRECT - Raw SQL only with placeholder bindings
$users = DB::select(
    'SELECT * FROM users WHERE name LIKE ?',
    ["%{$search}%"]
);
```

### Command injection

```php
// CRITICAL - User input passed to a shell
$filename = $request->input('filename');
exec("tar -czf /backups/{$filename}.tar.gz /data");
```

```php
// CORRECT - Validate, then use the Process facade (no shell interpolation)
use Illuminate\Support\Facades\Process;

public function backup(Request $request): JsonResponse
{
    $filename = $request->validated('filename');

    if (! preg_match('/^[a-zA-Z0-9_\-]+$/', $filename)) {
        return response()->json([
            'type'    => 'error',
            'code'    => 'backup::invalid_filename',
            'message' => __('backup::invalid_filename'),
        ], Response::HTTP_BAD_REQUEST);
    }

    // Array form runs without a shell, so metacharacters cannot escape
    $result = Process::run([
        'tar', '-czf', "/backups/{$filename}.tar.gz", '/data',
    ]);

    return $result->successful()
        ? response()->json(['type' => 'success', 'code' => 'backup::success'], Response::HTTP_OK)
        : response()->json(['type' => 'error', 'code' => 'backup::failed'], Response::HTTP_INTERNAL_SERVER_ERROR);
}
```

> `Illuminate\Support\Facades\Process` is available from the supported Laravel
> version. If you must call `exec()`, wrap every argument in `escapeshellarg()`.

---

## A04:2021 — Insecure Design

### What to check
- No rate limiting on sensitive endpoints (login, register, password reset).
- No account lockout after repeated failed logins.
- Weak password policy.
- Missing security headers.

Login rate limiting and failed-attempt tracking live in `api-auth-sanctum.md`.
Route-level `throttle` and named limiters live in `laravel-security.md`.

### Strong password policy

```php
use Illuminate\Validation\Rules\Password;

class RegistrationRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name'  => ['required', 'string', 'max:100'],
            'email' => ['required', 'email', 'max:255', 'unique:users,email'],
            'password' => [
                'required',
                'string',
                'confirmed',
                Password::min(12)
                    ->letters()
                    ->mixedCase()
                    ->numbers()
                    ->symbols()
                    ->uncompromised(), // Reject passwords seen in breach corpora
            ],
        ];
    }
}
```

### Security headers middleware

A reusable middleware that stamps hardening headers on every response.

```php
<?php
// vendor/module/src/Http/Middleware/SecurityHeaders.php

namespace Vendor\Module\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SecurityHeaders
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        // Prevent clickjacking
        $response->headers->set('X-Frame-Options', 'DENY');

        // Prevent MIME type sniffing
        $response->headers->set('X-Content-Type-Options', 'nosniff');

        // Content Security Policy (tighten per app)
        $response->headers->set('Content-Security-Policy', "default-src 'self'");

        // Force HTTPS for a year, including subdomains
        $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');

        // Limit referrer leakage
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');

        // Disable powerful features by default
        $response->headers->set('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');

        return $response;
    }
}
```

```php
// bootstrap/app.php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->append(\Vendor\Module\Http\Middleware\SecurityHeaders::class);
    })
    ->withExceptions(function (Exceptions $exceptions): void {
    })->create();
```

---

## A05:2021 — Security Misconfiguration

### What to check
- Debug mode enabled in production.
- Default credentials.
- Unnecessary features/providers enabled in production.
- Verbose error messages leaking internals.
- Directory listing enabled.
- Exposed config files (`.env`, `.git`, `composer.json`).

The full secure `.env` template and secrets guidance is in
`laravel-security.md`. Core config checks:

```php
// config/app.php

// WRONG - Debug hardcoded on
'debug' => true,

// CORRECT - Debug driven by env, defaults to off
'debug' => env('APP_DEBUG', false),
```

Disable development-only providers/tooling in production (profilers,
debug bars, request recorders). Install them under `require-dev` only and deploy
with `composer install --no-dev --optimize-autoloader`.

### Secure exception handling (no info leakage)

A reusable handler that returns JSON for API routes, hides internals in
production, and shows details only in `local`.

```php
<?php
// vendor/module/src/Exceptions/SecureExceptionHandler.php

namespace Vendor\Module\Exceptions;

use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpFoundation\Response;
use Throwable;

class SecureExceptionHandler
{
    public static function handleApiException(Request $request, Throwable $e): JsonResponse
    {
        $statusCode = self::getStatusCode($e);

        $response = [
            'type'    => 'error',
            'code'    => self::getErrorCode($e),
            'message' => self::getErrorMessage($e),
        ];

        if ($e instanceof ValidationException) {
            $response['errors'] = $e->errors();
        }

        // Only expose exception internals in local development
        if (app()->environment('local')) {
            $response['exception'] = [
                'message' => $e->getMessage(),
                'file'    => $e->getFile(),
                'line'    => $e->getLine(),
            ];
        }

        return response()->json($response, $statusCode);
    }

    protected static function getStatusCode(Throwable $e): int
    {
        return match (true) {
            $e instanceof ValidationException     => Response::HTTP_UNPROCESSABLE_ENTITY, // 422
            $e instanceof AuthenticationException => Response::HTTP_UNAUTHORIZED,         // 401
            $e instanceof AuthorizationException  => Response::HTTP_FORBIDDEN,            // 403
            $e instanceof ModelNotFoundException  => Response::HTTP_NOT_FOUND,            // 404
            method_exists($e, 'getStatusCode')    => $e->getStatusCode(),
            default                               => Response::HTTP_INTERNAL_SERVER_ERROR, // 500
        };
    }

    protected static function getErrorCode(Throwable $e): string
    {
        return match (true) {
            $e instanceof ValidationException     => 'validation.failed',
            $e instanceof AuthenticationException => 'auth.unauthenticated',
            $e instanceof AuthorizationException  => 'auth.unauthorized',
            $e instanceof ModelNotFoundException  => 'resource.not_found',
            default                               => 'error.'.strtolower(class_basename($e)),
        };
    }

    protected static function getErrorMessage(Throwable $e): string
    {
        // In production, return generic messages to avoid leaking internals
        if (app()->environment('production')) {
            return match (true) {
                $e instanceof ValidationException     => 'The given data was invalid.',
                $e instanceof AuthenticationException => 'Unauthenticated.',
                $e instanceof AuthorizationException  => 'This action is unauthorized.',
                $e instanceof ModelNotFoundException  => 'Resource not found.',
                default                               => 'An error occurred while processing your request.',
            };
        }

        // In local/staging, return the real message
        return $e->getMessage();
    }
}
```

```php
// bootstrap/app.php
use Vendor\Module\Exceptions\SecureExceptionHandler;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->append(\Vendor\Module\Http\Middleware\SecurityHeaders::class);
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        $exceptions->render(function (Throwable $e, $request) {
            if ($request->is('api/*')) {
                return SecureExceptionHandler::handleApiException($request, $e);
            }

            return null; // Let the framework handle non-API exceptions
        });
    })->create();
```

Example responses:

```json
// 422 Validation
{
    "type": "error",
    "code": "validation.failed",
    "message": "The given data was invalid.",
    "errors": {
        "email": ["The email field is required."],
        "password": ["The password must be at least 12 characters."]
    }
}

// 401 Authentication
{ "type": "error", "code": "auth.unauthenticated", "message": "Unauthenticated." }

// 403 Authorization
{ "type": "error", "code": "auth.unauthorized", "message": "This action is unauthorized." }

// 404 Not found
{ "type": "error", "code": "resource.not_found", "message": "Resource not found." }

// local only - includes debug info
{
    "type": "error",
    "code": "error.queryexception",
    "message": "SQLSTATE[42S02]: Base table or view not found",
    "exception": { "message": "...", "file": "/path/to/file.php", "line": 42 }
}
```

---

## A06:2021 — Vulnerable and Outdated Components

### What to check
- End-of-life Laravel/PHP release (treat as a finding).
- Dependencies with known CVEs.
- Unused dependencies.
- Missing security patches.

```bash
# Report known vulnerabilities in the lock file
composer audit

# Show outdated packages
composer outdated

# Update within your constraints
composer update
```

```jsonc
// composer.json - FINDING: pinning an unsupported, end-of-life major version
{
    "require": {
        "laravel/framework": "^8.0"
    }
}
```

```jsonc
// composer.json - keep constraints on the supported major version
// (see the "Supported versions" block in SKILL.md)
{
    "require": {
        "php": "^<supported>",
        "laravel/framework": "^<supported>",
        "laravel/sanctum": "^<supported>"
    }
}
```

Keep development-only tooling out of production:

```bash
composer install --no-dev --optimize-autoloader
```

```jsonc
// composer.json - profilers/debug bars/test frameworks belong here
{
    "require-dev": {
        "pestphp/pest": "^<supported>"
    }
}
```

---

## A07:2021 — Identification and Authentication Failures

### What to check
- Weak session management.
- Missing CSRF protection on web routes.
- Insecure token storage / no token expiration.
- Session fixation.
- No logout / no revocation of tokens.

Full Sanctum token lifecycle, session config and CSRF handling are covered in
`api-auth-sanctum.md` (tokens/sessions) and `laravel-security.md` (CSRF/XSS).
Summary:

- Encrypt sessions, `http_only`, `same_site=strict`, `secure` cookies in prod.
- Issue Sanctum tokens with an expiration; revoke on logout.
- Keep CSRF verification on stateful web routes; API routes authenticate with
  bearer tokens instead.

---

## A08:2021 — Software and Data Integrity Failures

### What to check
- Unsigned/unverified packages or repositories.
- Insecure deserialization of user input.
- Missing integrity checks on updates.
- Third-party scripts without Subresource Integrity (SRI).

```jsonc
// composer.json - trusted sources over HTTPS only
{
    "repositories": [
        { "type": "composer", "url": "https://packagist.org" }
    ],
    "config": {
        "preferred-install": "dist",
        "secure-http": true
    }
}
```

```php
// CRITICAL - Never unserialize user input (object-injection gadget chains)
$data = unserialize($request->input('data'));

// CORRECT - Use JSON and validate the shape
$data = $request->validate([
    'name'  => 'required|string',
    'value' => 'required|integer',
]);
```

---

## A09:2021 — Security Logging and Monitoring Failures

### What to check
- No logging of authentication events (success and failure).
- No audit trail for sensitive operations (delete, role change, exports).
- No suspicious-activity detection.

```php
use Illuminate\Support\Facades\Log;

// Log a failed login attempt
Log::channel('security')->warning('Failed login attempt', [
    'email'      => $email,
    'ip'         => $request->ip(),
    'user_agent' => $request->userAgent(),
    'timestamp'  => now(),
]);

// Audit a sensitive operation
Log::channel('audit')->info('User deleted', [
    'deleted_user_id' => $user->id,
    'deleted_by'      => auth()->id(),
    'ip'              => request()->ip(),
    'timestamp'       => now(),
]);
```

```php
// config/logging.php - dedicated, long-retention channels
'channels' => [
    'security' => [
        'driver' => 'daily',
        'path'   => storage_path('logs/security.log'),
        'level'  => 'info',
        'days'   => 90,
    ],
    'audit' => [
        'driver' => 'daily',
        'path'   => storage_path('logs/audit.log'),
        'level'  => 'info',
        'days'   => 365,
    ],
],
```

Never log secrets, raw passwords, tokens or full card numbers.

---

## A10:2021 — Server-Side Request Forgery (SSRF)

### What to check
- Unvalidated URLs fetched server-side from user input.
- Access to internal services (Redis, databases) or cloud metadata endpoints.

```php
// CRITICAL - Fetching an arbitrary user-supplied URL
$content = file_get_contents($request->input('url'));
// Attacker targets http://localhost:6379 or a cloud metadata endpoint
```

```php
// CORRECT - Validate scheme, block private IPs, optionally allow-list hosts
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Str;

public function fetchWebsite(Request $request): JsonResponse
{
    $url = $request->validated('url');
    $parsed = parse_url($url);

    if (! $parsed || ! isset($parsed['scheme'], $parsed['host'])) {
        return response()->json(['type' => 'error', 'code' => 'fetch::invalid_url'], Response::HTTP_BAD_REQUEST);
    }

    if (! in_array($parsed['scheme'], ['http', 'https'], true)) {
        return response()->json(['type' => 'error', 'code' => 'fetch::invalid_scheme'], Response::HTTP_BAD_REQUEST);
    }

    $ip = gethostbyname($parsed['host']);
    if ($this->isPrivateIP($ip)) {
        return response()->json(['type' => 'error', 'code' => 'fetch::private_ip'], Response::HTTP_FORBIDDEN);
    }

    $allowedDomains = config('app.allowed_fetch_domains', []);
    if (! empty($allowedDomains) && ! Str::endsWith($parsed['host'], $allowedDomains)) {
        return response()->json(['type' => 'error', 'code' => 'fetch::domain_not_allowed'], Response::HTTP_FORBIDDEN);
    }

    $response = Http::timeout(5)->get($url);

    return response()->json([
        'type' => 'success',
        'code' => 'fetch::success',
        'data' => ['content' => $response->body()],
    ], Response::HTTP_OK);
}

protected function isPrivateIP(string $ip): bool
{
    return filter_var(
        $ip,
        FILTER_VALIDATE_IP,
        FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE
    ) === false;
}
```
