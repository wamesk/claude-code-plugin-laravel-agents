---
name: wame-laravel-standards
description: "Use when writing or reviewing non-Nova Laravel code — REST API endpoints, controllers, services, managers, actions, jobs, migrations, models, enums, observers/listeners, API resources & collections, or Pest feature tests. Apply the standard API response envelope (type/code/data), ULID + dateTimeTz database conventions, the migrations-not-seeders rule for shipping data, the service-layer architecture, and translation-key patterns. Triggers on phrases like 'API response format', 'migration conventions', 'service layer', 'BaseCollection pagination', 'seed migration', 'observer registration', 'Pest feature test'."
---

# WAME Laravel Standards

Reusable, framework-level standards for building and reviewing **non-Nova** Laravel code in modular packages. This skill is the single source of truth for how APIs, databases, services and tests are structured across projects. Laravel Nova and browser (Dusk) content lives in a separate skill and is intentionally out of scope here.

## When to use

Load this skill whenever you:

- Design an API endpoint and need the response envelope, HTTP status codes, API Resources, or paginated Collections.
- Write a migration and need ULID keys, foreign keys, indexes, column-type choices, the `sort` column, or `dateTimeTz` datetimes.
- Need to ship default/reference/lookup data (use a seed **migration**, never a database seeder — see the rule below).
- Add business logic and must decide between a Service, Manager, or Action, or register an Observer/Listener.
- Write a Job or an Action class.
- Write Pest **feature** tests (API, service, action, auth, database) or set up test helpers, factories, or datasets.
- Add user-facing strings and need the translation-key convention.

## Supported versions

All version-specific guidance in this skill targets:

- **Laravel** 12.x (always the latest stable major)
- **PHP** 8.4+
- **Pest** 4.x (never PHPUnit class syntax)
- **Composer** for dependency management
- **Laravel Pint** for code style
- Attribute-based observer/factory registration requires the supported Laravel version (Laravel 11+).

Elsewhere in these docs, "the supported Laravel/Pest version" refers to the versions listed above.

## Namespace & package naming

Examples use the placeholder namespace root **`Vendor\Module`** and the composer package name **`vendor/module`**. The real vendor namespace root and per-module sub-namespace are defined **per-project in `CLAUDE.md`** — always resolve the actual namespace from there before writing code, and ask which module family a new module belongs to when it is ambiguous.

## Non-negotiables (quick list)

- `declare(strict_types = 1);` at the top of every PHP file; full type hints and return types everywhere.
- API responses use the `type` / `code` / `data|message` envelope and correct HTTP status codes.
- ULID primary keys by default; `dateTimeTz()` for **all** datetime columns including `created_at` / `updated_at` / `deleted_at` (never `timestamps()`, `softDeletes()`, or plain `datetime`/`timestamp`).
- Enums for boolean and static values; `sort` (not `order`) for ordering.
- Ship data/defaults via idempotent `*_seed_*.php` **migrations**, never database seeders. Factories exist for tests only.
- Business logic lives in Services/Managers/Actions — never in controllers. Controllers stay thin.
- No hardcoded user-facing strings — always translation keys. Key is always English words; value is the target language; one file per module/entity (see `reference/testing-patterns.md`).
- Pest syntax for tests; cover both success and error paths.

## Reference map

Detailed patterns, code examples and checklists live in `reference/`:

- [`reference/api-response-format.md`](reference/api-response-format.md) — success/error envelope, API Resources, `BaseCollection` pagination (`data`/`meta`/`links`), HTTP status codes.
- [`reference/database-conventions.md`](reference/database-conventions.md) — ULID keys, foreign keys & cascades, indexes, column-type optimization, enums, `sort` column, `dateTimeTz`, migration best practices, the migrations-not-seeders rule, and Laravel Sushi for static data.
- [`reference/service-layer.md`](reference/service-layer.md) — Service/Manager/Action patterns, constructor property promotion, Observers & Listeners with modern `#[ObservedBy]` / `#[UseFactory]` registration, Job & Action patterns, storage.
- [`reference/testing-patterns.md`](reference/testing-patterns.md) — Pest feature-test patterns (non-Nova), test helpers, factories, scenario builders, datasets, and the translation-key pattern.
