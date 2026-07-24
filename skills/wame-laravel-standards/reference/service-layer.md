# Service Layer, Observers, Jobs & Actions

Business logic never lives in controllers. It is organized into Services, Managers, and Actions, with side effects wired through Observers and Listeners. Placeholders `Vendor\Module` / `vendor/module` resolve to the real names in `CLAUDE.md`.

## Choosing a class type

- **Service** — cohesive read/write operations for one domain (fetching, paginating, creating, updating). The controller's main collaborator.
- **Manager** — orchestration and complex, multi-model calculations or lookups that go beyond a single simple operation.
- **Action** — one single-purpose operation, invoked via `handle()` (or `execute()`), reusable from controllers, jobs, or commands.

All three are plain classes resolved from the container, so they can be constructor-injected anywhere.

## Type safety

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Services;

use Illuminate\Support\Facades\Hash;
use Vendor\Module\Models\User;

class UserService
{
    // Full type hints and a return type on every method.
    public function create(string $name, string $email, string $password): User
    {
        return User::create([
            'name' => $name,
            'email' => $email,
            'password' => Hash::make($password),
        ]);
    }

    public function getAllPaginated(int $perPage = 15)
    {
        return User::query()
            ->with(['roles', 'profile']) // eager load to prevent N+1
            ->latest()
            ->paginate($perPage);
    }

    public function findById(string $id): User
    {
        return User::with(['roles', 'profile'])->findOrFail($id);
    }
}
```

## Constructor property promotion

Inject dependencies with promoted, `readonly` properties.

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Managers;

class PostManager
{
    public function __construct(
        private readonly PostService $postService,
        private readonly CommentManager $commentManager,
    ) {}
}
```

## Manager example

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Managers;

use Vendor\Module\Models\Post;

class PostManager
{
    /**
     * Fetch a post with its comments eager-loaded.
     */
    public function getPostBySlug(string $slug): ?Post
    {
        return Post::query()
            ->with(['comments.author'])
            ->where('slug', $slug)
            ->first();
    }

    /**
     * Count approved comments on a post.
     */
    public function countApprovedComments(Post $post): int
    {
        return $post->comments
            ->where('approved', true)
            ->count();
    }
}
```

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Managers;

use Vendor\Module\Models\Comment;

class CommentManager
{
    /**
     * Compute the next per-post sequence number, including soft-deleted rows.
     */
    public function getNextSequenceNumber(Comment $comment): int
    {
        return Comment::query()
            ->withTrashed()
            ->where('post_id', $comment->post_id)
            ->max('sequence_number') + 1;
    }
}
```

## Observers & Listeners (modern registration)

Prefer thin Observers that dispatch domain events, and Listeners that hold the reactive logic.

### Observer

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Observers;

use Vendor\Module\Events\Comment\CommentCreatingEvent;
use Vendor\Module\Events\Comment\CommentCreatedEvent;
use Vendor\Module\Events\Comment\CommentUpdatedEvent;
use Vendor\Module\Events\Comment\CommentDeletedEvent;
use Vendor\Module\Models\Comment;

class CommentObserver
{
    public function creating(Comment $comment): void
    {
        CommentCreatingEvent::dispatch($comment);
    }

    public function created(Comment $comment): void
    {
        CommentCreatedEvent::dispatch($comment);
    }

    public function updated(Comment $comment): void
    {
        CommentUpdatedEvent::dispatch($comment);
    }

    public function deleted(Comment $comment): void
    {
        CommentDeletedEvent::dispatch($comment);
    }
}
```

### Registering the Observer with an attribute

On the supported Laravel version, register the observer declaratively with `#[ObservedBy]` on the model — no ServiceProvider wiring needed.

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Models;

use Illuminate\Database\Eloquent\Attributes\ObservedBy;
use Illuminate\Database\Eloquent\Model;
use Vendor\Module\Observers\CommentObserver;

#[ObservedBy(CommentObserver::class)]
class Comment extends Model
{
    // ...
}
```

### Registering the Factory with an attribute

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Models;

use Illuminate\Database\Eloquent\Attributes\UseFactory;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Database\Eloquent\Model;
use Vendor\Module\Database\Factories\CommentFactory;

#[UseFactory(CommentFactory::class)]
class Comment extends Model
{
    protected static function newFactory(): Factory
    {
        return CommentFactory::new();
    }
}
```

### Listener (auto-discovered from the type hint)

Listeners need no registration — Laravel auto-discovers the event(s) from the `handle()` type hint (union types supported).

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Listeners\Comment;

use Vendor\Module\Events\Comment\CommentCreatingEvent;
use Vendor\Module\Managers\CommentManager;

class CommentSequenceNumberListener
{
    public function __construct(
        private readonly CommentManager $commentManager,
    ) {}

    public function handle(CommentCreatingEvent $event): void
    {
        if ($event->comment->sequence_number === null) {
            $event->comment->sequence_number = $this->commentManager
                ->getNextSequenceNumber($event->comment);
        }
    }
}
```

```php
// Union type-hint: one listener handling multiple events.
public function handle(CommentCreatingEvent|CommentUpdatingEvent $event): void
{
    // ...
}
```

**Why attributes + auto-discovery:** declarative (visible on the model), no ServiceProvider boilerplate, type-safe.

## Job pattern

Async work goes into queued jobs with promoted, `readonly` constructor state.

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class PostPublishJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        private readonly string $postId,
    ) {}

    public function handle(): void
    {
        // Job logic here.
    }
}
```

## Action pattern

A single-purpose operation exposed through `handle()`.

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Actions;

use SimpleSoftwareIO\QrCode\Facades\QrCode;
use Vendor\Module\Models\Post;

class PostQrAction
{
    public function handle(Post $post): string
    {
        return QrCode::format('svg')
            ->size(200)
            ->generate($post->id);
    }
}
```

## File storage

Store files on S3-compatible object storage / CDN via the `s3` disk. Configure the disk in `config/filesystems.php` and reference it by name — never hardcode a provider.

```php
<?php

declare(strict_types = 1);

namespace Vendor\Module\Services;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

class MediaService
{
    public function upload(UploadedFile $file, string $path = 'uploads'): string
    {
        $filePath = Storage::disk('s3')->putFile($path, $file);

        return Storage::disk('s3')->url($filePath);
    }

    public function delete(string $path): bool
    {
        return Storage::disk('s3')->delete($path);
    }
}
```

## Checklist

- ✅ `declare(strict_types = 1);`, full type hints and return types.
- ✅ Business logic in Services/Managers/Actions — never in controllers.
- ✅ Constructor property promotion with `readonly` for injected dependencies.
- ✅ Eager-load relationships to prevent N+1.
- ✅ Thin Observers dispatch events; Listeners hold reactive logic.
- ✅ Register observers/factories via `#[ObservedBy]` / `#[UseFactory]`; let listeners auto-discover.
- ✅ Async work in queued Jobs; single-purpose operations in Actions.
- ✅ Files on the `s3` disk (S3-compatible object storage / CDN), never a hardcoded provider.
