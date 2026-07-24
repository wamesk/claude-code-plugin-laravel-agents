# File Storage & CDN Optimization

Serve user-uploaded and static files from S3-compatible object storage fronted by a CDN — never from the application server's local disk. This keeps the app tier stateless, offloads bandwidth, and puts assets close to users.

Namespaces use a generic `Vendor\Module` placeholder; the real namespace is defined per-project in `CLAUDE.md`.

---

## 1. Disk configuration

Configure an `s3` disk in `config/filesystems.php`. This works with any S3-compatible object storage / CDN provider — set the `endpoint` for non-AWS providers and leave it empty for AWS S3.

```php
'disks' => [

    's3' => [
        'driver'                  => 's3',
        'key'                     => env('AWS_ACCESS_KEY_ID'),
        'secret'                  => env('AWS_SECRET_ACCESS_KEY'),
        'region'                  => env('AWS_DEFAULT_REGION'),
        'bucket'                  => env('AWS_BUCKET'),
        // Endpoint for S3-compatible object storage (leave empty for AWS S3).
        'endpoint'                => env('AWS_ENDPOINT'),
        'url'                     => env('AWS_URL'),
        'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),

        // CDN base URL placed in front of the bucket, e.g. https://cdn.example.com
        'cdn_url'                 => env('AWS_CDN_URL'),

        // Upload defaults: long-lived cache + public read.
        'visibility'              => 'public',
        'options'                 => [
            'CacheControl' => 'max-age=31536000, public', // 1 year
        ],
    ],

],
```

---

## 2. Serve assets through the CDN

Return the CDN URL for stored files — it is served from an edge cache, which is far faster than hitting the origin bucket directly.

```php
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class MediaService
{
    public function upload(UploadedFile $file, string $path = 'uploads'): array
    {
        $filename = Str::random(40) . '.' . $file->getClientOriginalExtension();

        // Store on the S3-compatible disk with public visibility.
        $filePath = Storage::disk('s3')->putFileAs($path, $file, $filename, 'public');

        // Prefer the CDN URL; fall back to the origin bucket URL if no CDN is set.
        $cdnUrl = config('filesystems.disks.s3.cdn_url');
        $url = $cdnUrl
            ? rtrim($cdnUrl, '/') . '/' . $filePath
            : Storage::disk('s3')->url($filePath);

        return [
            'path'      => $filePath,
            'url'       => $url,
            'size'      => $file->getSize(),
            'mime_type' => $file->getMimeType(),
        ];
    }
}
```

---

## 3. Optimization checklist

- **Cache headers** — set a long `CacheControl` (`max-age`) on immutable assets so browsers and the CDN cache aggressively. Use content-hashed filenames (as above, a random/hashed name) so a changed file gets a new URL and cache-busting is automatic.
- **Public visibility** — mark public assets `public` on upload so the CDN can cache and serve them without signed requests.
- **CDN, not origin** — always hand out the `cdn_url`-based URL to clients; reserve direct bucket URLs for server-side operations.
- **Optimize images** — resize/compress and generate responsive variants (thumbnails) before or on upload; prefer modern formats (WebP/AVIF) where clients support them. Offload heavy processing to a queue job (see [caching-strategies.md](caching-strategies.md)).
- **Keep files off the app server** — no user uploads on the local/app disk; the app tier stays stateless and horizontally scalable.
- **Stream large downloads** — for exports/large files, stream from the disk rather than loading the whole file into memory.
