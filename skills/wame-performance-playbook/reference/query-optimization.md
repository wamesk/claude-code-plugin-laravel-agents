# Query Optimization

Database access is the single largest source of Laravel performance problems. This file covers N+1 prevention, eager-loading variants, column selection, indexing, pagination, streaming large datasets, API Resource pitfalls, connection tuning, and how to measure query cost.

All migration examples use `dateTimeTz()` for `created_at` / `updated_at` / `deleted_at` — never `timestamps()`, `softDeletes()`, or plain `datetime`.

---

## 1. N+1 query problem

### Detection & fix

```php
// BAD — N+1 query problem
// Scenario: display 100 users with their profiles.
public function index(): JsonResponse
{
    $users = User::all(); // 1 query

    // N+1 problem: 1 + 100 queries (one per user) = 101 queries!
    foreach ($users as $user) {
        echo $user->profile->bio; // 100 additional queries
    }

    return response()->json([
        'data' => UserResource::collection($users),
    ]);
}
// Impact: 101 queries, ~500ms, scales linearly with row count.

// GOOD — eager loading (2 queries total)
public function index(): JsonResponse
{
    $users = User::with('profile')->get(); // users + profiles

    return response()->json([
        'type' => 'success',
        'code' => 'user::user.index.success',
        'data' => UserResource::collection($users),
    ], Response::HTTP_OK);
}
// Impact: 2 queries (~50x fewer), ~50ms, constant regardless of row count.
```

### Complex / nested eager loading

```php
// BAD — multiple N+1 problems stacked
$posts = Post::all(); // 1 query
foreach ($posts as $post) {
    $post->author->name;     // N queries
    $post->category->name;   // N queries
    $post->comments;         // N queries
    foreach ($post->comments as $comment) {
        $comment->user->name; // N*M queries
    }
}
// 100 posts × 10 comments => 1 + 100 + 100 + 100 + 1000 = 1301 queries!

// GOOD — nested eager loading with a constrained relation
$posts = Post::query()
    ->with([
        'author',
        'category',
        'comments' => fn ($query) => $query->latest()->limit(5), // only last 5
        'comments.user',
    ])
    ->latest()
    ->paginate(20);
// 5 queries total (posts, authors, categories, comments, comment users) vs 1301.
```

### Conditional eager loading

Load relationships only when the caller asks for them.

```php
public function show(Request $request, string $id): JsonResponse
{
    $query = Post::query();

    if ($request->boolean('include_author')) {
        $query->with('author');
    }

    if ($request->boolean('include_comments')) {
        $query->with([
            'comments' => fn ($q) => $q->latest()->limit(10),
            'comments.user',
        ]);
    }

    if ($request->boolean('include_related')) {
        $query->with('category.posts');
    }

    $post = $query->findOrFail($id);

    return response()->json([
        'type' => 'success',
        'code' => 'post::post.show.success',
        'data' => PostResource::make($post),
    ], Response::HTTP_OK);
}

// GET /api/v1/posts/123                                   -> 1 query
// GET /api/v1/posts/123?include_author=true               -> 2 queries
// GET /api/v1/posts/123?include_author=true&include_comments=true -> 4 queries
```

### Lazy eager loading

Load relationships after the parent is already fetched (e.g. inside a controller branch).

```php
$post = Post::findOrFail($id);

$post->load([
    'author'   => fn ($query) => $query->select('id', 'name', 'email'),
    'comments' => fn ($query) => $query->latest()->limit(5),
]);
```

---

## 2. Select only the columns you need

Avoid `SELECT *` when large text/blob columns are not required.

```php
// BAD
$users = User::all(); // SELECT * FROM users

// GOOD
$users = User::query()
    ->select('id', 'name', 'email', 'created_at')
    ->get();

// GOOD — with a trimmed relationship (always include the FK column)
$users = User::query()
    ->select('id', 'name', 'email')
    ->with(['profile' => fn ($query) => $query->select('id', 'user_id', 'bio', 'avatar')])
    ->get();
```

Benefits: less data over the wire, lower memory, faster serialization. When selecting columns on a relationship, always include the foreign key Laravel needs to match rows (`user_id` above).

---

## 3. Database indexing

Missing indexes turn `WHERE`/`ORDER BY` into full table scans. Add indexes for the query patterns the app actually runs.

```php
// BEFORE — no indexes: every filter is a full table scan
Schema::create('posts', function (Blueprint $table) {
    $table->ulid('id')->primary();
    $table->foreignUlid('user_id')->constrained();
    $table->foreignUlid('category_id')->constrained();
    $table->string('title', 255);
    $table->enum('status', ['draft', 'published', 'archived']);
    $table->dateTimeTz('published_at')->nullable();
    $table->unsignedInteger('views')->default(0);
    $table->dateTimeTz('created_at');
    $table->dateTimeTz('updated_at');
    $table->dateTimeTz('deleted_at')->nullable();
    // No indexes — slow WHERE status, WHERE user_id + status, ORDER BY published_at.
});

// AFTER — indexed for the real query patterns
Schema::create('posts', function (Blueprint $table) {
    $table->ulid('id')->primary();
    $table->foreignUlid('user_id')->constrained('users')->cascadeOnUpdate()->cascadeOnDelete();
    $table->foreignUlid('category_id')->constrained('categories')->cascadeOnUpdate()->nullOnDelete();
    $table->string('title', 255);
    $table->enum('status', [PostStatusEnum::DRAFT, PostStatusEnum::PUBLISHED, PostStatusEnum::ARCHIVED])
        ->default(PostStatusEnum::DRAFT);
    $table->dateTimeTz('published_at')->nullable();
    $table->unsignedInteger('views')->default(0);
    $table->unsignedInteger('sort')->default(0);
    $table->dateTimeTz('created_at');
    $table->dateTimeTz('updated_at');
    $table->dateTimeTz('deleted_at')->nullable();

    // Single-column indexes
    $table->index('status');        // WHERE status = 'published'
    $table->index('published_at');  // ORDER BY published_at
    $table->index('created_at');    // ORDER BY created_at
    $table->index('deleted_at');    // soft-delete queries

    // Composite indexes for common patterns
    $table->index(['user_id', 'status']);        // a user's published posts
    $table->index(['category_id', 'status']);    // a category's published posts
    $table->index(['status', 'published_at']);   // latest published posts
    $table->index(['user_id', 'created_at']);    // a user's latest posts
    $table->index(['status', 'views']);          // most-viewed published posts
});
// A single-column index seek can be ~5ms vs ~2000ms for a full scan on 1M rows.
```

### Index strategy rules

1. **Single-column indexes** — foreign keys, status/flag columns, date columns used in `ORDER BY`, and any frequently filtered column.
2. **Composite indexes — order matters.** Put the most selective / equality-filtered column first, the range/sort column last. `index(['status', 'published_at'])`:
   - `WHERE status = 'x' ORDER BY published_at` — uses the index.
   - `WHERE status = 'x'` — uses the index.
   - `ORDER BY published_at` alone — may not use the index.
3. **Don't over-index.** Every index slows `INSERT`/`UPDATE`/`DELETE`. Create indexes only for actual query patterns and drop unused ones.
4. **Verify with `EXPLAIN`.** `EXPLAIN SELECT * FROM posts WHERE status = 'published';` — confirm an index is used, not a full scan.

```php
// A 3-column composite index that serves this exact query:
$table->index(['user_id', 'status', 'published_at']);

Post::query()
    ->where('user_id', $userId)
    ->where('status', 'published')
    ->orderBy('published_at', 'desc')
    ->get();
```

---

## 4. Pagination

Never return an unbounded result set.

```php
// BAD — loads every row into memory
$posts = Post::all(); // 100k rows -> 500MB+, 5s+

// GOOD — standard pagination (includes total count)
$posts = Post::query()
    ->with(['author', 'category'])
    ->latest()
    ->paginate(20);

return response()->json([
    'type' => 'success',
    'code' => 'post::post.index.success',
    'data' => PostResource::collection($posts),
    'meta' => [
        'current_page' => $posts->currentPage(),
        'last_page'    => $posts->lastPage(),
        'per_page'     => $posts->perPage(),
        'total'        => $posts->total(),
    ],
    'links' => [
        'first' => $posts->url(1),
        'last'  => $posts->url($posts->lastPage()),
        'prev'  => $posts->previousPageUrl(),
        'next'  => $posts->nextPageUrl(),
    ],
], Response::HTTP_OK);
```

**Cursor pagination** — best for infinite scroll and deep pages; uses `WHERE id > last_id` instead of `OFFSET`, so it stays fast regardless of depth.

```php
$posts = Post::query()
    ->with(['author', 'category'])
    ->latest('id')
    ->cursorPaginate(20);

return response()->json([
    'type' => 'success',
    'code' => 'post::post.index.success',
    'data' => PostResource::collection($posts),
    'meta' => [
        'next_cursor' => $posts->nextCursor()?->encode(),
        'prev_cursor' => $posts->previousCursor()?->encode(),
    ],
], Response::HTTP_OK);
```

**Simple pagination** — when the total count is not needed; skips the `COUNT(*)` query and is faster than `paginate()`.

```php
$posts = Post::query()
    ->with(['author', 'category'])
    ->latest()
    ->simplePaginate(20);
```

---

## 5. Streaming large datasets

For exports, batch jobs, and reports, never call `all()` on a large table.

```php
// BAD — 100k rows loaded at once (~500MB, crashes)
$users = User::all();
foreach ($users as $user) { /* ... */ }

// GOOD — chunk(): process a fixed number of rows per round trip
User::query()
    ->orderBy('id')
    ->chunk(1000, function ($users) {
        foreach ($users as $user) {
            // export logic; memory freed after each chunk
        }
    });

// GOOD — lazy(): a LazyCollection that streams rows in batches under the hood
User::query()
    ->orderBy('id')
    ->lazy(1000)
    ->each(fn ($user) => /* export logic */);

// GOOD — cursor(): one row at a time, minimal memory (single connection held open)
foreach (User::cursor() as $user) {
    // export logic
}
```

### Lazy collections with transformations

`LazyCollection` keeps memory flat (~constant) even through `filter`/`map` pipelines.

```php
User::query()
    ->orderBy('id')
    ->lazy()
    ->filter(fn (User $user) => $user->is_active)
    ->map(fn (User $user) => [
        'name'  => $user->name,
        'email' => $user->email,
    ])
    ->each(fn (array $userData) => $this->exportData($userData));
```

Rule of thumb: `chunk()` when you mutate rows and want them committed per batch; `lazy()`/`cursor()` when you only read/stream.

---

## 6. API Resource optimization

Resources silently trigger queries when they touch un-loaded relationships or call `->count()`.

```php
// BAD — every accessor is a hidden query
class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'      => $this->id,
            'title'   => $this->title,
            'content' => $this->content,
            'author'  => [                     // N+1 if author not eager loaded
                'name'  => $this->author->name,
                'email' => $this->author->email,
            ],
            'comments_count' => $this->comments()->count(), // extra query each row
            'likes_count'    => $this->likes()->count(),    // extra query each row
        ];
    }
}

// GOOD — whenLoaded() + withCount() computed in the query
class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'title'      => $this->title,
            'content'    => $this->content,
            'created_at' => $this->created_at->toIso8601String(),
            'author'     => $this->whenLoaded('author', fn () => [
                'id'   => $this->author->id,
                'name' => $this->author->name,
            ]),
            'comments_count' => $this->comments_count ?? 0,
            'likes_count'    => $this->likes_count ?? 0,
        ];
    }
}

// Controller — load relations and counts up front
$posts = Post::query()
    ->with('author')
    ->withCount(['comments', 'likes']) // both counts in one query
    ->latest()
    ->paginate(20);
// 3 queries total: posts, authors, counts.
```

---

## 7. Database connection tuning

Configure the connection for production throughput in `config/database.php`.

```php
'mysql' => [
    'driver'    => 'mysql',
    'host'      => env('DB_HOST', '127.0.0.1'),
    'port'      => env('DB_PORT', '3306'),
    'database'  => env('DB_DATABASE', 'forge'),
    'username'  => env('DB_USERNAME', 'forge'),
    'password'  => env('DB_PASSWORD', ''),
    'charset'   => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'strict'    => true,

    'options' => [
        // Persistent connections reduce per-request connect overhead.
        PDO::ATTR_PERSISTENT => true,
        // Buffered queries let the server free result resources sooner.
        PDO::MYSQL_ATTR_USE_BUFFERED_QUERY => true,
        // Fail fast if the DB is unreachable.
        PDO::ATTR_TIMEOUT => 5,
    ],
],
```

Note: true connection pooling is provided by the runtime/proxy layer (e.g. an async server or a DB proxy), not by the PDO driver. Persistent connections are the portable win for classic FPM deployments.

---

## 8. DB performance testing

Measure query count and cost — don't guess.

```php
// Count queries around a code path
DB::enableQueryLog();
// ... run the request / call the service
$queries = DB::getQueryLog();
count($queries); // target: < 15 per request

// Assert in a Pest test that a payload stays under a query budget
it('lists posts without N+1', function () {
    DB::enableQueryLog();

    $this->getJson('/api/v1/posts')->assertOk();

    expect(count(DB::getQueryLog()))->toBeLessThanOrEqual(5);
});

// Peak memory (target: < 50MB)
memory_get_peak_usage() / 1024 / 1024;
```

Confirm index usage with `EXPLAIN` on the raw SQL, and use the profiling tools in
[caching-strategies.md](caching-strategies.md) (Telescope / Debugbar / the performance middleware) to catch slow queries (> 50ms) in real traffic.
