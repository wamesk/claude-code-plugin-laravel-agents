# Caching Strategies

Once queries are lean (see [query-optimization.md](query-optimization.md)), caching removes repeat work. This file covers Redis-backed caching (query/model/response/tags/computed), Laravel's production caches, offloading long work to queue jobs, profiling, the optimization checklist, and the audit report template.

Namespaces use a generic `Vendor\Module` placeholder; the real namespace is defined per-project in `CLAUDE.md`.

---

## 1. Query result caching (Redis)

```php
use Illuminate\Support\Facades\Cache;

// BAD — hits the database on every request (~200ms)
public function index(): JsonResponse
{
    $categories = Category::query()->with('posts')->get();

    return response()->json([
        'data' => CategoryResource::collection($categories),
    ]);
}

// GOOD — cache the result; subsequent requests read from Redis (~5ms)
public function index(): JsonResponse
{
    $categories = Cache::remember(
        'categories.all',
        now()->addHours(24),
        fn () => Category::query()->with('posts')->get(),
    );

    return response()->json([
        'type' => 'success',
        'code' => 'category::category.index.success',
        'data' => CategoryResource::collection($categories),
    ], Response::HTTP_OK);
}

// Invalidate on write
public function update(UpdateCategoryRequest $request, string $id): JsonResponse
{
    $category = Category::findOrFail($id);
    $category->update($request->validated());

    Cache::forget('categories.all');

    return response()->json([
        'type' => 'success',
        'code' => 'category::category.update.success',
        'data' => CategoryResource::make($category),
    ], Response::HTTP_OK);
}
```

---

## 2. Model caching

```php
class UserService
{
    public function findById(string $id): User
    {
        return Cache::remember(
            "users.{$id}",
            now()->addHour(),
            fn () => User::with('profile')->findOrFail($id),
        );
    }

    public function update(string $id, array $data): User
    {
        $user = User::findOrFail($id);
        $user->update($data);

        Cache::forget("users.{$id}"); // invalidate the single entry

        return $user;
    }
}
```

---

## 3. Response caching (HTTP payload)

Cache the fully-assembled response array for read-heavy, rarely-changing endpoints.

```php
public function show(string $id): JsonResponse
{
    $cacheKey = "api.posts.{$id}";

    if (Cache::has($cacheKey)) {
        return response()->json(Cache::get($cacheKey), Response::HTTP_OK);
    }

    $post = Post::with(['author', 'category'])->findOrFail($id);

    $response = [
        'type' => 'success',
        'code' => 'post::post.show.success',
        'data' => PostResource::make($post),
    ];

    Cache::put($cacheKey, $response, now()->addHour());

    return response()->json($response, Response::HTTP_OK);
}
// First request ~150ms, cached requests ~2ms.
```

---

## 4. Cache tags (grouped invalidation)

Tags let you flush a whole group without knowing every key. Requires a taggable store (Redis / Memcached).

```php
public function getUserPosts(string $userId): Collection
{
    return Cache::tags(['users', "user:{$userId}"])->remember(
        "user.{$userId}.posts",
        now()->addHour(),
        fn () => Post::where('user_id', $userId)->get(),
    );
}

public function clearUserCache(string $userId): void
{
    Cache::tags(["user:{$userId}"])->flush(); // just this user's entries
}

public function clearAllUsersCache(): void
{
    Cache::tags(['users'])->flush(); // every user-tagged entry
}
```

---

## 5. Computed value caching (persist derived values)

When an accessor runs an expensive computation on every read, compute it once on save and store it in a column instead.

```php
// BAD — recomputed on every access
class Post extends Model
{
    public function getWordCountAttribute(): int
    {
        return str_word_count(strip_tags($this->content));
    }

    public function getReadingTimeAttribute(): int
    {
        return (int) ceil($this->word_count / 200); // 200 wpm
    }
}

// GOOD — compute on save, read from a column
class Post extends Model
{
    protected $fillable = ['title', 'content', 'word_count', 'reading_time'];

    protected static function booted(): void
    {
        static::saving(function (Post $post) {
            if ($post->isDirty('content')) {
                $post->word_count   = str_word_count(strip_tags($post->content));
                $post->reading_time = (int) ceil($post->word_count / 200);
            }
        });
    }
}
```

The supporting columns are added by an ordinary schema migration (this is derived state, not seed data):

```php
Schema::table('posts', function (Blueprint $table) {
    $table->unsignedInteger('word_count')->default(0)->after('content');
    $table->unsignedInteger('reading_time')->default(0)->after('word_count');

    $table->index('word_count');
    $table->index('reading_time');
});
```

> Reference data and defaults are a different concern: ship those via idempotent `*_seed_*` **migrations** whose `up()` writes rows only when absent — never via database seeders. Static/lookup data may use a read-only source such as Laravel Sushi. Factories remain for tests only.

---

## 6. Laravel production caches

Precompile config, routes, views, and events for a faster, lighter boot.

```bash
# Enable in production (after deploy)
php artisan config:cache   # merged config
php artisan route:cache    # compiled route table
php artisan view:cache     # precompiled Blade
php artisan event:cache    # cached event/listener map

# One-shot production optimize (also does autoloader-friendly caching)
php artisan optimize

# Boot time can drop from ~300ms to ~50ms and memory from ~40MB to ~25MB.

# Clear after each deploy / config change
php artisan optimize:clear
# or individually: config:clear, route:clear, view:clear, event:clear
```

Do not enable `config:cache` in local development — it ignores subsequent `.env` edits until cleared.

---

## 7. Queue jobs for long operations

Move anything slow (bulk mail, exports, third-party calls, image processing) out of the request lifecycle.

```php
// BAD — the request blocks while 10k emails send (times out)
public function sendNewsletter(Request $request): JsonResponse
{
    $users = User::where('subscribed', true)->get();
    foreach ($users as $user) {
        Mail::to($user)->send(new NewsletterMail($newsletter));
    }
    return response()->json(['success' => true]);
}

// GOOD — dispatch and return immediately (~50ms)
use Vendor\Module\Jobs\SendNewsletterJob;

public function sendNewsletter(Request $request): JsonResponse
{
    $newsletter = Newsletter::findOrFail($request->input('newsletter_id'));

    SendNewsletterJob::dispatch($newsletter);

    return response()->json([
        'type'    => 'success',
        'code'    => 'newsletter::newsletter.queued',
        'message' => __('newsletter::newsletter.queued'),
    ], Response::HTTP_ACCEPTED);
}
```

```php
// The job chunks internally so the worker stays within memory.
class SendNewsletterJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public Newsletter $newsletter,
    ) {}

    public function handle(): void
    {
        User::query()
            ->where('subscribed', true)
            ->chunk(100, function ($users) {
                foreach ($users as $user) {
                    Mail::to($user)->send(new NewsletterMail($this->newsletter));
                }
            });
    }
}
```

Run workers (`php artisan queue:work` / Horizon on Redis), handle failed jobs, and monitor the queue depth.

---

## 8. Monitoring & profiling

### Laravel Telescope (development)

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
# Access at /telescope
```

Watch for: slow queries (> 50ms), N+1 patterns, per-request memory, cache hit/miss, queue job timing.

### Laravel Debugbar (development)

```bash
composer require barryvdh/laravel-debugbar --dev
```

Shows query count and time, memory usage, view render time, and route info inline.

### Laravel Pulse (production)

Use Pulse for real-time production metrics (slow queries, slow requests, queue throughput, exceptions) without the overhead of Telescope.

### Custom performance middleware

Log slow requests and expose per-request metrics in local dev.

```php
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class PerformanceMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        DB::enableQueryLog();

        $startTime   = microtime(true);
        $startMemory = memory_get_usage();

        $response = $next($request);

        $executionTime = (microtime(true) - $startTime) * 1000;          // ms
        $memoryUsage   = (memory_get_usage() - $startMemory) / 1024 / 1024; // MB
        $queryCount    = count(DB::getQueryLog());

        if ($executionTime > 500) {
            Log::channel('performance')->warning('Slow request detected', [
                'url'            => $request->fullUrl(),
                'method'         => $request->method(),
                'execution_time' => round($executionTime, 2) . 'ms',
                'memory_usage'   => round($memoryUsage, 2) . 'MB',
                'query_count'    => $queryCount,
                'user_id'        => $request->user()?->id,
            ]);
        }

        if (app()->environment('local')) {
            $response->headers->set('X-Execution-Time', round($executionTime, 2) . 'ms');
            $response->headers->set('X-Memory-Usage', round($memoryUsage, 2) . 'MB');
            $response->headers->set('X-Query-Count', (string) $queryCount);
        }

        return $response;
    }
}
```

---

## 9. Optimization checklist

**Database queries**
- [ ] No N+1 problems; eager loading for all touched relationships
- [ ] Indexes on WHERE/JOIN columns; composite indexes for common query patterns
- [ ] `SELECT` only needed columns
- [ ] Pagination on list endpoints (< 100 rows/page)
- [ ] Chunking / lazy collections for large datasets
- [ ] Query execution < 50ms; max 10–15 queries per request

**Caching**
- [ ] Redis configured; query/model/response caching where it pays off
- [ ] Cache invalidation strategy defined (keys or tags)
- [ ] Cache hit ratio > 80%
- [ ] `config:cache` / `route:cache` / `view:cache` enabled in production

**API**
- [ ] Simple queries < 100ms, complex < 500ms
- [ ] API Resources used (not raw models); `whenLoaded` / `withCount`
- [ ] Conditional eager loading; response compression enabled

**Memory**
- [ ] < 50MB per request, peak < 256MB
- [ ] Lazy collections / chunking for large sets; no leaks

**Queue jobs**
- [ ] Long operations queued; workers running; failed jobs handled and monitored

**File storage** (see [file-storage-cdn.md](file-storage-cdn.md))
- [ ] CDN configured; cache headers set; images optimized; files off the app server

**Production**
- [ ] `composer install --optimize-autoloader --no-dev`
- [ ] `config:cache`, `route:cache`, `view:cache`, `event:cache`
- [ ] OPcache on; HTTP/2 on; gzip/br compression on

---

## 10. Performance audit report template

Use this structure when reporting an audit. State concrete before/after numbers — never "faster" without figures.

```markdown
## Performance Audit Report

**Project:** {project_name}
**Endpoint:** GET /api/v1/posts
**Current performance:** Slow (2.5s response time)

### Current metrics

| Metric           | Current | Target  | Status   |
|------------------|---------|---------|----------|
| Response time    | 2500ms  | < 500ms | CRITICAL |
| Database queries | 156     | < 15    | CRITICAL |
| Memory usage     | 180MB   | < 50MB  | HIGH     |
| Cache hit ratio  | 0%      | > 80%   | HIGH     |

### Critical issues

#### 1. N+1 query disaster (155 queries)
**File:** `PostController.php:25`
**Impact:** ~2000ms query time, ~150MB memory

Current:
    $posts = Post::all();
    foreach ($posts as $post) {
        $post->author->name;        // 100 queries
        $post->category->name;      // 50 queries
        $post->comments->count();   // 5 queries
    }

Optimized:
    $posts = Post::query()
        ->with(['author', 'category'])
        ->withCount('comments')
        ->paginate(20);

Gain: queries 156 -> 3, response 2000ms -> 50ms, memory 150MB -> 10MB.

#### 2. Missing pagination
Loading 10,000 rows at once. Fix: `Post::paginate(20)`.
Gain: memory 180MB -> 5MB, response 500ms -> 50ms.

#### 3. No caching
Every request hits the database. Fix:
    $posts = Cache::remember('posts.page.' . $page, now()->addMinutes(15),
        fn () => Post::with(['author', 'category'])->withCount('comments')->paginate(20));
Gain: cached requests ~5ms.

### Expected results

| Metric           | Before | After | Improvement    |
|------------------|--------|-------|----------------|
| Response time    | 2500ms | 50ms  | 50x faster     |
| Database queries | 156    | 3     | 52x fewer      |
| Memory usage     | 180MB  | 5MB   | 36x lower      |
| Cache hit ratio  | 0%     | 85%   | new            |

### Implementation priority
- Immediate (< 1 day): eager loading (N+1), pagination, indexes.
- This week: caching strategy; enable Laravel production caches.
- This month: performance monitoring; sweep remaining slow endpoints.

### Verification
- Count queries: `DB::enableQueryLog(); ... count(DB::getQueryLog());` (< 15).
- Measure response time (e.g. `curl -w` against the endpoint).
- Check Telescope/Pulse for queries > 50ms.
- `memory_get_peak_usage() / 1024 / 1024` (< 50MB).
```

---

## Do / don't

**Always:** eager load (`with()`); paginate; index WHERE/JOIN columns; cache hot data with a clear invalidation plan; `select()` specific columns; profile with Telescope/Debugbar/Pulse; queue long operations; measure before and after.

**Never:** ignore N+1; `all()` on large tables; query inside loops; skip indexes; load relationships you don't use; ship data/defaults through database seeders (use `*_seed_*` migrations instead).
