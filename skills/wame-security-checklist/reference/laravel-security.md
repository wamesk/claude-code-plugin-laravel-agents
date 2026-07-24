# Laravel-Specific Security

Mass assignment, file uploads, CSRF/XSS, rate limiting, CORS, secrets/env, the
pre-production audit checklist and the audit report template.

> Any `vendor/module/...` path and `Vendor\Module` namespace below is a
> placeholder — the real namespace is defined per-project in `CLAUDE.md`.

---

## Mass Assignment Protection

```php
// CRITICAL - $guarded = [] plus create($request->all()) lets a user set any
// column, e.g. is_admin=1 or role_id
class User extends Model
{
    protected $guarded = [];
}

User::create($request->all());
```

```php
// CORRECT - Explicit fillable, hidden and casts; persist validated data only
class User extends Model
{
    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'password'          => 'hashed',
    ];
}

User::create($request->validated());
```

Rules of thumb:
- Never `create()`/`update()` with `$request->all()`; use `$request->validated()`.
- Keep sensitive columns (`is_admin`, `role_id`, `verified_at`, balances) out of
  `$fillable`; set them explicitly in a service after an authorization check.
- Add secrets and tokens to `$hidden` so they never leak through API resources.

---

## File Upload Security

```php
// CRITICAL - No validation; an attacker can upload an executable script
public function upload(Request $request): JsonResponse
{
    $path = $request->file('file')->store('uploads');

    return response()->json(['path' => $path]);
}
```

```php
// CORRECT - Validate type/size, randomize the name, store off the web root
use Illuminate\Support\Str;
use Illuminate\Support\Facades\Log;

public function upload(UploadFileRequest $request): JsonResponse
{
    $file = $request->file('file');

    // Random name; keep only a validated extension
    $filename = Str::random(40).'.'.$file->getClientOriginalExtension();

    // 'private' disk is NOT web-accessible; serve later via a signed/controlled route
    $path = $file->storeAs('uploads', $filename, 'private');

    Log::channel('security')->info('File uploaded', [
        'filename'      => $filename,
        'original_name' => $file->getClientOriginalName(),
        'mime_type'     => $file->getMimeType(),
        'size'          => $file->getSize(),
        'user_id'       => auth()->id(),
        'ip'            => request()->ip(),
    ]);

    return response()->json([
        'type' => 'success',
        'code' => 'upload::success',
        'data' => ['path' => $path, 'filename' => $filename],
    ], Response::HTTP_CREATED);
}
```

```php
// UploadFileRequest.php - validate both mimes (extension) and mimetypes (content)
class UploadFileRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'file' => [
                'required',
                'file',
                'max:10240', // 10 MB
                'mimes:jpg,jpeg,png,pdf,doc,docx',
                'mimetypes:image/jpeg,image/png,application/pdf,application/msword',
            ],
        ];
    }
}
```

Checklist for uploads:
- Validate extension (`mimes`) AND actual content (`mimetypes`).
- Store outside the public directory; never trust the client filename.
- The upload directory must not be executable by the web server.
- For public delivery, front the private disk with S3-compatible object
  storage / CDN and hand out short-lived signed URLs — see
  `../../wame-performance-playbook/reference/` for file-storage patterns.

---

## CSRF and XSS

### CSRF

```php
// Web (stateful) routes keep CSRF verification (default).
Route::middleware(['web'])->group(function () {
    Route::post('/contact', [ContactController::class, 'store']); // CSRF token required
});

// API routes authenticate with bearer tokens, so CSRF is excluded for them.
Route::middleware(['auth:sanctum'])->group(function () {
    Route::post('/users', [UserController::class, 'store']); // Bearer token, no CSRF
});
```

CSRF is excluded for `api/*` in the CSRF middleware configuration; never widen
that exclusion to stateful web routes.

### XSS (output escaping)

```blade
{{-- SAFE - Blade escapes by default --}}
<p>{{ $comment->body }}</p>

{{-- DANGEROUS - unescaped; only for trusted, sanitized HTML --}}
<div>{!! $comment->body !!}</div>
```

- Prefer `{{ }}`; reserve `{!! !!}` for HTML you have sanitized server-side
  (e.g. an HTML Purifier pass).
- Never build DOM from raw user input in the frontend; if the API returns
  user-generated HTML, sanitize before rendering.
- Set a Content-Security-Policy header (see the SecurityHeaders middleware in
  `owasp-top10.md`) to reduce the blast radius of any injected script.

---

## Rate Limiting

```php
// routes/api.php - global API limiter
Route::middleware(['throttle:api'])->group(function () {
    Route::get('/users', [UserController::class, 'index']);
});

// Stricter, dedicated limiter for sensitive endpoints
Route::middleware(['throttle:login'])->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
});
```

```php
// Named limiters (a service provider boot method)
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});

RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->ip());
});
```

Apply strict limits to login, registration, password reset, OTP verification and
any expensive/export endpoint. Per-user login lockout logic is in
`api-auth-sanctum.md`.

---

## CORS

```php
// config/cors.php
return [
    'paths' => ['api/*'],

    'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE'],

    // WRONG - wildcard origin (especially with credentials) is a finding
    // 'allowed_origins' => ['*'],

    // CORRECT - explicit allow-list
    'allowed_origins' => [
        'https://example.com',
        'https://app.example.com',
    ],

    'allowed_origins_patterns' => [
        '/^https:\/\/.*\.example\.com$/', // Trusted subdomains only
    ],

    'allowed_headers' => ['Content-Type', 'Authorization', 'X-Requested-With'],

    'exposed_headers' => [],

    'max_age' => 3600,

    'supports_credentials' => true,
];
```

`allowed_origins => ['*']` together with `supports_credentials => true` is
invalid and unsafe — always enumerate trusted origins.

---

## Secrets and Environment Configuration

```bash
# FINDING - insecure production configuration
APP_DEBUG=true
APP_ENV=local
```

```bash
# Secure production configuration
APP_DEBUG=false
APP_ENV=production
APP_URL=https://example.com

# Session security
SESSION_DRIVER=redis
SESSION_LIFETIME=120
SESSION_SECURE_COOKIE=true
SESSION_HTTP_ONLY=true
SESSION_SAME_SITE=strict

# Database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=prod_database
DB_USERNAME=prod_user
DB_PASSWORD=<strong-random-password>

# Cache / queue
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis

# Redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=<strong-random-password>
REDIS_PORT=6379
```

Secrets management rules:
- Read secrets through `config()` (backed by `env()`), never call `env()` outside
  config files — config caching wipes runtime `env()` reads.
- No hardcoded keys, tokens or passwords anywhere in the codebase.
- `.env` must be git-ignored and file mode `600`.
- A strong `APP_KEY` must be generated (`php artisan key:generate`).
- Rotate any credential that has ever been committed to version control.

---

## Pre-Production Security Audit Checklist

```yaml
Environment & Configuration:
  - [ ] APP_DEBUG=false in production
  - [ ] APP_ENV=production
  - [ ] Strong APP_KEY generated
  - [ ] HTTPS enforced (SESSION_SECURE_COOKIE=true)
  - [ ] Secure session configuration (encrypt, http_only, same_site)
  - [ ] Security headers middleware registered
  - [ ] CORS restricted to trusted origins
  - [ ] .env git-ignored and not web-accessible

Authentication & Authorization:
  - [ ] Sanctum configured with token expiration
  - [ ] Rate limiting on login/register/reset
  - [ ] Failed login attempt tracking / lockout
  - [ ] Strong password policy enforced
  - [ ] Policies/Gates implemented
  - [ ] Authorization (ownership) checks on every endpoint
  - [ ] CSRF protection enabled for stateful web routes

Database Security:
  - [ ] No raw SQL with interpolation (use Eloquent/bindings)
  - [ ] Mass assignment protection ($fillable, validated data)
  - [ ] Passwords hashed with bcrypt/argon2id
  - [ ] Sensitive data encrypted at rest
  - [ ] Database credentials secured; no default credentials

API Security:
  - [ ] Protected routes behind auth middleware
  - [ ] Rate limiting configured
  - [ ] Input validation via Form Requests
  - [ ] API Resources used (no raw models returned)
  - [ ] Correct HTTP status codes
  - [ ] Error messages leak nothing sensitive

File Security:
  - [ ] Upload validation (type, size, mime + mimetype)
  - [ ] Safe (random) filename generation
  - [ ] Files stored outside the public directory
  - [ ] No execute permission on the upload directory

Dependencies:
  - [ ] composer audit passes
  - [ ] Dependencies on a supported major version
  - [ ] No unused dependencies
  - [ ] Dev dependencies excluded from production build

Logging & Monitoring:
  - [ ] Security events logged
  - [ ] Failed authentication attempts logged
  - [ ] Sensitive operations audited
  - [ ] Log rotation/retention configured
  - [ ] No secrets written to logs

Code Quality:
  - [ ] No hardcoded secrets
  - [ ] No debug code / dump() / var_dump()
  - [ ] No commented-out security checks
  - [ ] Input validation everywhere
  - [ ] Output escaping (XSS prevention)
```

---

## Audit Report Template

Structure security-audit feedback like this. Order findings by severity
(Critical -> High -> Medium -> Low), then positives.

```markdown
## Security Audit Report

**Project:** {project_name}
**Audit Date:** {date}
**Severity:** Critical issues found / Warnings / Secure

---

### CRITICAL VULNERABILITIES (Fix immediately)

#### 1. SQL Injection in UserController
- **Severity:** Critical (CVSS 9.8)
- **File:** `vendor/module/src/Http/Controllers/UserController.php:45`
- **Issue:** Direct SQL concatenation with user input
- **Attack Vector:** Arbitrary SQL execution
- **Impact:** Complete database compromise

**Vulnerable code:**
` ``php
$users = DB::select("SELECT * FROM users WHERE name = '{$name}'");
` ``

**Exploitation:**
` ``
Input:  test' OR '1'='1
Result: SELECT * FROM users WHERE name = 'test' OR '1'='1' -> returns all users
` ``

**Remediation:**
` ``php
$users = User::where('name', $name)->get();
` ``

---

### HIGH RISK (Fix within a week)

#### 2. Missing Authorization Check (IDOR)
- **Severity:** High (CVSS 7.5)
- **File:** `vendor/module/src/Http/Controllers/UserController.php:30`
- **Issue:** Any authenticated user can access any user's data

**Remediation:**
` ``php
$this->authorize('view', $user);
` ``

---

### MEDIUM RISK (Fix within a month)

#### 3. Weak Password Policy
- **Severity:** Medium (CVSS 5.3)
- **Issue:** Minimum password length is only 8 characters
- **Recommendation:** Require 12+ characters with mixed case, numbers, symbols

---

### LOW RISK (Best practices)

#### 4. Missing Security Headers
- **Recommendation:** Register the SecurityHeaders middleware

---

### POSITIVE FINDINGS

- HTTPS properly configured
- CSRF protection enabled
- Sanctum authentication properly implemented
- Rate limiting configured

---

### OWASP Top 10 Coverage

- [x] A01 - Broken Access Control: 2 issues found
- [x] A02 - Cryptographic Failures: Secure
- [x] A03 - Injection: 1 critical SQL injection
- [x] A04 - Insecure Design: Secure
- [x] A05 - Security Misconfiguration: 1 issue found
- [x] A06 - Vulnerable Components: Secure
- [x] A07 - Authentication Failures: Secure
- [x] A08 - Data Integrity Failures: Secure
- [x] A09 - Logging Failures: Secure
- [x] A10 - SSRF: Secure

---

### Priority Action Items

1. **IMMEDIATE (24 hours):** Fix SQL injection; add authorization checks.
2. **THIS WEEK:** Register security headers; review all user-input handling.
3. **THIS MONTH:** Strengthen password policy; run penetration testing.

---

### Recommendations

1. Log all authentication events to a dedicated security channel.
2. Add automated dependency/security scanning to CI/CD.
3. Schedule regular audits (quarterly).
4. Train the team on secure coding practices.
```
