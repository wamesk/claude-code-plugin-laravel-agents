---
name: wame-performance-playbook
description: "Use when optimizing Laravel application performance — diagnosing or fixing N+1 queries, adding database indexes, choosing eager/lazy/conditional loading, paginating or chunking large datasets, designing caching strategies (Redis, query cache, response cache, config/route/view caches, cache tags), moving long operations to queue jobs, optimizing API Resources (withCount/whenLoaded), tuning database connections, optimizing file storage and CDN delivery, profiling with Telescope/Debugbar/Pulse, or producing a performance audit report with before/after metrics."
---

# WAME Performance Playbook

Reference material for diagnosing and fixing Laravel performance bottlenecks: slow queries, high memory usage, and long response times. The goal is measurable improvement — every change should come with a before/after metric.

Namespaces in the examples use a generic `Vendor\Module` placeholder; the real namespace is defined per-project in `CLAUDE.md`.

## When to use

- A request runs slowly, spikes memory, or issues far too many queries (suspected N+1).
- Deciding between eager loading, conditional/lazy eager loading, and column selection.
- Adding or reviewing database indexes for common query patterns.
- Choosing a data-fetching strategy for large sets: pagination, chunking, lazy collections, cursors.
- Designing a caching layer (query/model/response cache, cache tags) or invalidation strategy.
- Enabling production caches (config/route/view/event) and offloading work to queue jobs.
- Optimizing API Resources so they don't trigger hidden queries.
- Optimizing file storage / CDN delivery on S3-compatible object storage.
- Setting up profiling (Telescope, Debugbar, Pulse) or writing a performance audit report.

## Performance targets (rules of thumb)

- Simple GET < 100ms; complex GET with joins < 500ms; writes < 200ms.
- Max 10–15 queries per request; each query < 50ms; indexes used on all WHERE/JOIN columns.
- Memory < 50MB per request, peak < 256MB.
- Cache hit ratio > 80%.

## Supported versions

- Laravel 12.x
- PHP 8.2+
- MySQL 8.x / PostgreSQL 15.x
- Cache & queue driver: Redis
- Profiling: Laravel Telescope, Laravel Debugbar, Laravel Pulse
- Optional HTTP cache layer: Varnish / Cloudflare

Where a version is not stated in the prose below, assume the supported Laravel version listed here.

## Reference files

- [reference/query-optimization.md](reference/query-optimization.md) — N+1 prevention and eager loading (complex, conditional, lazy), column selection, database indexing and index-strategy rules, pagination, chunking, lazy collections and cursors, API Resource optimization, database connection tuning, and DB performance testing.
- [reference/caching-strategies.md](reference/caching-strategies.md) — Redis-backed query/model/response caching, cache tags, computed-value caching, Laravel production caches (config/route/view/event), queue jobs for long operations, monitoring & profiling (Telescope/Debugbar/custom middleware), the full optimization checklist, and the audit response-format template.
- [reference/file-storage-cdn.md](reference/file-storage-cdn.md) — file storage and CDN optimization on S3-compatible object storage: disk configuration, CDN URLs for static assets, cache headers, and upload tuning.
