# API Response Format

Every API endpoint returns a consistent envelope so clients can branch on a single, predictable shape. The namespace root `Vendor\Module` and package name `vendor/module` are placeholders — resolve the real ones from `CLAUDE.md`.

## Success envelope

```php
use Symfony\Component\HttpFoundation\Response;

return response()->json([
    'type' => 'success',
    'code' => 'module::entity.action.success',
    'data' => ResourceClass::make($data),
], Response::HTTP_OK);
```

- `type` — always `success` or `error`.
- `code` — a translation key (see the translation-key pattern in `testing-patterns.md`), not a human sentence.
- `data` — an API Resource, a Collection, or `null`.

## Error envelope

```php
use Symfony\Component\HttpFoundation\Response;

return response()->json([
    'type' => 'error',
    'code' => 'module::entity.action.error',
    'message' => __('module::entity.action.error'),
    'exception' => app()->environment('local') ? $e->getMessage() : null,
], Response::HTTP_BAD_REQUEST);
```

- `message` — the human-readable, translated message.
- `exception` — the raw exception message **only** in the `local` environment; `null` everywhere else so internals never leak in production.

## API Resources

Never return an Eloquent model directly — always transform it through a Resource.

### Single resource

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            // Conditionally include relationships only when eager-loaded.
            'author' => AuthorResource::make($this->whenLoaded('author')),
            // Conditionally include fields.
            'published_at' => $this->when($this->published_at !== null, $this->published_at),
        ];
    }
}
```

Key helpers:

- `whenLoaded('relation')` — include a relationship only if it was eager-loaded (prevents N+1).
- `when($condition, $value)` — include a field conditionally.

## Collections & pagination (BaseCollection)

For any paginated list, extend the shared `BaseCollection` so pagination metadata is emitted in a standardized `data` / `meta` / `links` shape. `BaseCollection` lives in the shared base package (`vendor/module` — resolve the real base package from `CLAUDE.md`).

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Http\Resources;

use Vendor\Base\Http\Resources\BaseCollection;

class PostCollection extends BaseCollection
{
    public $collects = PostResource::class;
}
```

Returning it from a controller:

```php
return response()->json([
    'type' => 'success',
    'code' => 'module::post.index.success',
    'data' => new PostCollection($posts), // $posts is a LengthAwarePaginator
], Response::HTTP_OK);
```

Resulting response structure:

```json
{
  "data": [],
  "meta": {
    "total": 100,
    "count": 15,
    "per_page": 15,
    "current_page": 1,
    "last_page": 7
  },
  "links": {
    "first": "https://example.com/api/v1/posts?page=1",
    "last": "https://example.com/api/v1/posts?page=7",
    "prev": null,
    "next": "https://example.com/api/v1/posts?page=2"
  }
}
```

- `meta` carries pagination counters (`total`, `count`, `per_page`, `current_page`, `last_page`).
- `links` carries navigation URLs (`first`, `last`, `prev`, `next`).
- Always pass a `LengthAwarePaginator` (e.g. from `->paginate()`) to the collection so `meta`/`links` are populated.

## HTTP status codes

Use the `Symfony\Component\HttpFoundation\Response` constants, never bare integers.

| Constant | Code | Meaning |
| --- | --- | --- |
| `Response::HTTP_OK` | 200 | Record found / successful operation |
| `Response::HTTP_CREATED` | 201 | Record created |
| `Response::HTTP_BAD_REQUEST` | 400 | Validation failed |
| `Response::HTTP_UNAUTHORIZED` | 401 | Not authenticated |
| `Response::HTTP_FORBIDDEN` | 403 | Authenticated but not permitted |
| `Response::HTTP_NOT_FOUND` | 404 | Record not found |
| `Response::HTTP_INTERNAL_SERVER_ERROR` | 500 | Internal server error |

## Controller shape

Controllers stay thin — HTTP wiring only. Validation goes in FormRequests, business logic in Services/Managers/Actions (see `service-layer.md`).

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Http\Controllers\Api\v1;

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Vendor\Module\Http\Requests\v1\StorePostRequest;
use Vendor\Module\Http\Resources\PostResource;
use Vendor\Module\Services\PostService;

class PostController extends Controller
{
    public function __construct(
        private readonly PostService $postService,
    ) {}

    public function store(StorePostRequest $request): JsonResponse
    {
        try {
            $post = $this->postService->create($request->validated());

            return response()->json([
                'type' => 'success',
                'code' => 'module::post.store.success',
                'data' => PostResource::make($post),
            ], Response::HTTP_CREATED);
        } catch (\Throwable $e) {
            return response()->json([
                'type' => 'error',
                'code' => 'module::post.store.error',
                'message' => __('module::post.store.error'),
                'exception' => app()->environment('local') ? $e->getMessage() : null,
            ], Response::HTTP_INTERNAL_SERVER_ERROR);
        }
    }
}
```

## Checklist

- ✅ `type` + `code` on every response; `data` for success, `message` for errors.
- ✅ Translation key in `code`, translated string in `message`.
- ✅ `exception` exposed only in `local`.
- ✅ Correct `Response::HTTP_*` constant.
- ✅ Resources for all output; `whenLoaded`/`when` for conditional fields.
- ✅ `BaseCollection` for paginated lists (`data`/`meta`/`links`).
- ✅ FormRequests for validation; controllers stay thin.
