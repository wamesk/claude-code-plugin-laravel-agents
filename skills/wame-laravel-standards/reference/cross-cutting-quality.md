# Cross-cutting Quality — UI/UX, Performance, Security, Reachability, Framework

Four dimensions that the acceptance criteria of a task almost never mention, and
that QA checks anyway — plus a fifth, `framework`: write the idioms of the
framework and language versions the project actually has installed, not the ones
remembered from older releases. The keys below are the same ones the
`teamwork-task-test` plugin uses in its review step (`ui_ux`, `performance`,
`security`, `reachability`, `framework`), so a finding raised at QA time names the
same rule you were supposed to apply at build time. Build it in, rather than
getting it back as a finding. QA and review treat `framework` as **advisory**
(recommendations, never a failed criterion); on the build side it is a rule for
the code you write or change.

This file holds **build-time rules** (what to do while writing the change) and a
**pre-finish self-check**. It is not an audit procedure — for a deep dive use
`wame-security-checklist` or `wame-performance-playbook`. For Laravel Nova
screens (resources, lenses, dashboards, tools, `Nova::mainMenu()`) use the
`wame-nova-patterns` skill from the `laravel-nova-agents` plugin instead of the
reachability section below; it also holds the Nova 4 vs Nova 5 differences.

Keep it proportionate: each dimension says when it applies. A pure backend
change (a job, a service, a JSON endpoint with no screen) does not grow UI
boilerplate — `ui_ux` and `reachability` simply do not apply to it. `framework`
applies to every line you write or change, and to nothing you do not touch.

## `reachability` — every new screen is reachable by clicking

**Applies when** the change adds, renames, or removes a screen: a web route that
renders a view, a Livewire full-page component, an Inertia page, an SPA route.
JSON API endpoints are not screens — the screen that consumes them is.

A page that exists, is authorized, and renders fine, but that no menu item and no
link points at, is found by typing the URL — which means it is found by nobody.
`php artisan route:list` shows what is **registered**; only the rendered
navigation shows what is **reachable**.

Rules, in the same change as the new screen:

1. **Menu entry.** Add it to the navigation the sibling screens use, visible to
   the roles that should use the screen. Find where siblings live by grepping for
   a neighbour's route name, e.g. `grep -rn "route('orders.index')" resources/`.
   - **Blade / Livewire:** the layout's nav partial or component (e.g.
     `layouts/navigation.blade.php`, a `<x-nav-link>`); Livewire links use
     `wire:navigate` like their neighbours.
   - **Inertia:** the layout component under `resources/js/Layouts/`; the
     permission that shows the item travels as a shared prop from
     `HandleInertiaRequests::share()`.
   - **SPA:** the router entry *and* the nav component; the route guard mirrors
     the backend ability, but the API is the real boundary.
2. **Inbound links ("prekliky")** from the screens where a user would naturally
   look for it: index row → detail, parent detail → its child list, related
   records → each other, a button/action on a detail view, a breadcrumb back up.
3. **Menu visibility and route authorization agree.** Wrap the link in the same
   ability that guards the route — `@can('viewAny', Invoice::class)` around the
   link, `->can('viewAny', Invoice::class)` (or `authorize()` in the controller /
   Livewire `mount()`) on the route. *Visible-but-403* is a UX defect;
   *hidden-but-open* is a security defect.
4. **Named routes, not literal paths.** `route('invoices.index')` in Blade throws
   when the name disappears, so a rename breaks a test instead of leaving a
   silent 404 in the menu.
5. **Renames and removals leave no dead links.** Grep the old route name *and*
   the old path across `resources/`, `routes/`, `app/`, and front-end sources;
   update every menu entry, link, redirect, and breadcrumb in the same change.
6. **Deliberately URL-only pages are named as such** — a signed e-mail deep
   link, a public landing page, a legal text linked from an e-mail. Say so in the
   commit/PR and list it in `teamwork-task-test`'s `allow_orphans`, so QA does
   not report it as an orphan.

## `security` — every new entry point is gated like its neighbours

**Applies when** the change adds or alters a route, controller action, Livewire
action, form, button that triggers a write, query, job input, or config/secret.

- Put the new route in the same middleware group as its neighbours (`auth`,
  `verified`, tenant middleware) **plus** an object-scoped policy check —
  `$this->authorize('update', $invoice)`, `->can('update', 'invoice')`, or a
  FormRequest `authorize()` that checks *this* record. A role check any tenant
  passes is not authorization (IDOR). Nested bindings use `scopeBindings()`.
- Tenant / company scope is not bypassed: no `DB::table()`, raw join, or
  `withoutGlobalScopes()` that skips the scope the model normally carries.
- Validate through a FormRequest; keep explicit `$fillable`; pass
  `$request->validated()` — never `$request->all()` — into `fill()` / `create()`.
- No interpolation into raw SQL (bind `?` parameters), shell (`Process::run()`
  with an array), or a sort/filter column taken from input (whitelist it).
- Crafted input gets a handled answer — a validation error, `findOrFail()` →
  404, a policy → 403 — never an unhandled 500. Anything the user must read
  travels as a validation error or flash message: with `APP_DEBUG=false` a thrown
  exception's message becomes a generic 500 page. Never leak paths, SQL, or other
  tenants' IDs.
- No secrets in code, logs, commit messages, or reports.
- **Hiding a menu item is not access control.** The route must refuse on its own.

Deep dive: `wame-security-checklist` → `reference/api-auth-sanctum.md`
(ownership / IDOR) and `reference/owasp-top10.md`.

## `performance` — the patterns that only hurt at scale

**Applies when** the change reads or writes the database, renders a list, runs a
batch/migration, or calls something slow.

- No query inside a loop; eager-load every relation the view or resource touches
  per row (`with()`, `withCount()`). `Model::preventLazyLoading(! app()->isProduction())`
  in a service provider turns a missed eager load into a failing test.
- Paginate every list (`paginate()` / `cursorPaginate()`); never `->get()` an
  unbounded table into a view.
- Batch jobs and data migrations use `chunkById()` / `lazyById()`, not a
  hydrated whole table.
- Index new foreign keys and the columns the new screen filters or sorts by;
  check existing indexes first (`php artisan db:table <table>`) so you do not add
  a duplicate.
- Filter in SQL (`where`), not in PHP (`->get()->filter()`).
- Slow work (mail, PDF, export, external API) goes to a queued job; the screen
  shows the "queued" state.
- Cache only with an invalidation story (who forgets the key, and when).
- Name the row count at which a pattern starts to hurt — "fine at 2 000 rows, a
  full scan at 200 000" is a decision; "might be slow" is not.

Deep dive: `wame-performance-playbook` → `reference/query-optimization.md`.

## `ui_ux` — UI/UX and accessibility

**Applies when** the change renders something a person sees: a Blade / Livewire /
Inertia / SPA view, a component, an e-mail or notification. For a pure backend
change only the translation rule applies (to validation and flash messages).

- Follow the sibling screens: same layout, components, spacing, table/form
  patterns. Reuse the existing component before writing a new one.
- Every interactive element has an accessible name — an icon-only button gets an
  `aria-label` (translated).
- A disabled control carries a real `disabled` or `aria-disabled="true"` **and**
  a visible reason (`title` / `aria-describedby`). A greyed-out control with no
  reason is a support ticket; a bare `<span>` standing in for a button is
  announced as plain text.
- Form controls have labels; validation errors sit next to their field; images
  have `alt`.
- State is never conveyed by colour alone — pair it with text or an icon.
- Design the loading, empty, and error states: an empty list says so (and offers
  the next action when the user may take it), a slow action shows progress, a
  failure shows a message instead of a blank page.
- Destructive actions ask for confirmation and report the result.
- No layout shift — reserve space for images and late-loading content.
- Every user-facing string goes through `__()` with an **English key** in the
  module's own lang file (one file per module/entity; the shipped languages are
  listed in the project's `CLAUDE.md`). A new key must resolve to text —
  `__($key) !== $key`. Never make a key both a string and a prefix: with nested
  lang arrays (and in i18next) a string at `invoice.status` makes
  `invoice.status.hint` unreachable and the screen shows the raw key. Put hints
  in their own group — `hint.status`, not `status.hint`. The
  `field.{name}.label` / `field.{name}.help` shape in `testing-patterns.md`
  already follows this rule.

## `framework` — the installed versions' idioms, not remembered ones

**Applies when** the change writes or changes PHP, Laravel, or Pest code — to the
new and changed lines only. Code the task does not touch is out of scope (see the
guardrails below).

Model memory is stale in both directions. It suggests patterns the installed
version has replaced (a `$casts` property on Laravel 12, `uses()->in()` on
Pest 3), and it suggests features the installed version does not have — which is
not a style issue but a bug: a property hook on PHP 8.3 is a parse error
(`syntax error, unexpected token "{"`), `once()` on Laravel 10 is an undefined
function, and a `casts()` method on Laravel 10 is never called, so the attributes
are silently not cast.

### 1. Detect the real versions first

Once per task, before writing code; reuse the result. The snippets are
zsh-safe:

```bash
# Locked versions of the framework and its companions
jq -r '[.packages[], (."packages-dev" // [])[]]
  | map(select(.name | test("^(laravel/(framework|boost|nova)|pestphp/pest|livewire/livewire|inertiajs/inertia-laravel)$")))
  | .[] | "\(.name) \(.version)"' composer.lock

# The PHP floor the code must run on
jq -r '"php require: \(.require.php // "-"), config.platform.php: \(.config.platform.php // "-")"' composer.json

# Front-end, only when the change touches a view (ranges; the exact version is in the JS lock file)
jq -r '(.dependencies // {}) + (.devDependencies // {}) | to_entries[]
  | select(.key | test("^(vue|react|tailwindcss|vite|typescript|@inertiajs/.+)$")) | "\(.key) \(.value)"' package.json
# ...and the browser target that gates JS/CSS features
jq -c '.browserslist // "no browserslist key in package.json"' package.json
if [ -f .browserslistrc ]; then cat .browserslistrc; fi
```

- **PHP floor.** Code against the lowest PHP the project admits:
  `config.platform.php` when it is set, otherwise the lowest version
  `require.php` allows (`^8.2` → 8.2, even when your machine and production run
  8.4), unless the project's `CLAUDE.md` names the production version. A package
  or module with a wide range — `wame-package-development` ships
  `illuminate/support: ^10.0 | ^11.0 | ^12.0 | ^13.0` and `php: ^8.2` — must run
  on the **lowest** version in that range, minor included.
- **Laravel Boost.** With `laravel/boost` installed, its `application-info` tool
  lists the installed package versions. Its `php_version` is the PHP that runs the
  MCP server, not the project's floor — take the floor from `composer.json`.
- **Version blocks in these skills are targets, not licences.** The "Supported
  versions" sections name what the examples were written for. The installed
  version decides what the code may use; when an example needs a newer version
  than the project has, write the older equivalent.

### 2. Look the version up in current docs, not in memory

In this order:

1. **Laravel Boost `search-docs`** (`mcp__laravel-boost__search-docs`) when the
   project has `laravel/boost` — it searches the docs of the *installed* versions
   of Laravel, Pest, Livewire, Inertia, Nova, Tailwind and more; pass `packages`
   to narrow it.
2. **context7** — `resolve-library-id`, then `query-docs` with the entry that
   matches the installed major.
3. **Official docs** via WebFetch — `https://laravel.com/docs/<major>.x/<page>`
   together with that major's `upgrade` and `releases` pages,
   `https://pestphp.com/docs/<page>`, `https://www.php.net/releases/<version>/en.php`.

Before reaching for something "new", check the installed major's upgrade guide
and release notes for what it added and what it deprecated. When a choice was
not obvious, name the page you checked in your summary. Where docs and memory
disagree, `vendor/laravel/framework/src` is the final word — read it, never edit
it.

### 3. Prefer the current idiom and the built-in feature

Examples to illustrate, not a checklist — every row is gated by rule 1. The
version is the minimum that has the feature; `10.0` means "already in Laravel
10.0" (older majors were not checked). Checked in 2026-09 against the framework
sources and changelogs (Laravel 10–13, Pest 2–4) and the php.net release notes.

**PHP**

| Min | Use | Instead of |
|-----|-----|-----------|
| 8.1 | Backed enums with methods, interfaces, and constants | Class constants plus a static label map |
| 8.1 | `readonly` promoted properties; `new` in initializers — parameter defaults only, e.g. a promoted `__construct(private Clock $clock = new SystemClock())`; as a property default (`private Clock $clock = new SystemClock();`) it is a fatal error on every PHP version | Mutable value objects; `?Clock $clock = null` plus `??=` in the body |
| 8.1 | First-class callables — `strlen(...)`, `$this->format(...)` | `'strlen'` strings, `[$this, 'format']`, `Closure::fromCallable()` |
| 8.2 | `readonly class` for DTOs and value objects | `readonly` repeated on every property |
| 8.3 | `#[\Override]` on overriding methods; typed class constants (`const string PREFIX = 'inv-';`); `json_validate()` | A silent orphan when the parent method is renamed; untyped constants; `json_decode()` only to validate |
| 8.4 | Property hooks (`public string $name { set => trim($value); }`), asymmetric visibility (`public private(set) int $count`), `array_find()` / `array_any()` / `array_all()` | Getter/setter pairs; `foreach` loops that only search |

**Laravel**

| Min | Use | Instead of |
|-----|-----|-----------|
| 10.0 | `Attribute::make(get: …, set: …)` accessors; enum casts; `Rule::enum()`; `$request->enum()` | `getFooAttribute()` / `setFooAttribute()`; `in:` rules and a manual `tryFrom()` |
| 10.0 | `Model::shouldBeStrict(! app()->isProduction())` in a service provider (lazy loading, silently discarded and missing attributes) | Finding N+1 and attribute typos in production |
| 10.0 | `ShouldBeUnique`; job middleware (`WithoutOverlapping`, `RateLimited`, `ThrottlesExceptions`) | Hand-rolled cache locks inside `handle()` |
| 10.0 | `Password::defaults()`; `Http::retry()` / `Http::pool()`; `chunkById()` / `lazyById()`; `->scopeBindings()`; `HasUlids` | Password rules repeated per FormRequest; retry loops with `sleep()`; sequential HTTP calls; offset chunking; manual parent checks; ULIDs generated by hand |
| 10.33 | `Number::format()` / `currency()` / `percentage()` / `fileSize()` — all need the `intl` extension (`fileSize()` formats through `format()`) | `number_format()` string building |
| 10.44 | `#[ObservedBy]`, `#[ScopedBy]` on the model | Registering observers and global scopes in a provider or `booted()` |
| 10.47 | `whereAll()` / `whereAny()` (`whereNone()` from 11.19) | A nested `where(fn …)` with one `orWhere` per column |
| 11.0 | `casts()` method; `once()`; `Context::add()` for data shared by logs and queued jobs; app configuration in `bootstrap/app.php` (`withRouting()` / `withMiddleware()` / `withExceptions()`) | The `$casts` property; a static memo property; a trace ID passed through every call; `app/Http/Kernel.php` — but an app upgraded from 10 may keep its kernels: register where the project already does |
| 11.23 | `defer()` for small work after the response (import `Illuminate\Support\defer`, 11.24+); `Cache::flexible()` (stale-while-revalidate); `Concurrency::run()` for independent slow calls (beta in the 11.x docs; an app upgraded from 10 may need `ConcurrencyServiceProvider`) | Slow recomputation on the request path; sequential independent calls. Work that must not be lost is still a queued job — deferred functions skip 4xx/5xx responses unless `->always()` |
| 11.28 / 11.39 | `#[CollectedBy]` / `#[UseFactory]` | `newCollection()` / `newFactory()` overrides |
| 12.4 | `#[Scope]` on a `protected` method | The `scopeFoo()` prefix |
| 12.18 | `#[UsePolicy]` on the model | `Gate::policy()` for a policy outside the conventional location |
| 13.0 | Model attributes (`#[Table]`, `#[Fillable]`, `#[Hidden]` …), controller `#[Middleware]` / `#[Authorize]`, job `#[Tries]` / `#[Backoff]` / `#[Timeout]` — Laravel 13 requires PHP 8.3 | The equivalent properties; follow what the sibling classes use |

Not something to switch on as a side effect of a task:
`Model::automaticallyEagerLoadRelationships()` (12.8) is marked beta in the 12.x
docs. Explicit `with()` stays the rule (see `performance`).

**Pest**

| Min | Use |
|-----|-----|
| Pest 2 | `arch()` tests, e.g. `arch('modules use strict types')->expect('Vendor\Module')->toUseStrictTypes()` — on Pest 2 the description is required (`arch()` without one throws an `ArgumentCountError`); it is optional from Pest 3 |
| Pest 3 | Arch presets — `arch()->preset()->php()` / `->laravel()` / `->security()`; the `pest()->extends(…)->use(…)->in(…)` configuration API instead of `uses()->in()`; mutation testing with `covers()` and `--mutate` |
| Pest 4 | Browser testing with `visit()` — needs `pestphp/pest-plugin-browser` |

**Front-end** (Blade components, Livewire, Inertia, Vue, Tailwind): the same rule
— take the versions from `composer.lock` / `package.json` and look the idiom up
the same way (Boost `search-docs` covers Livewire, Inertia, and Tailwind). JS and
CSS features are also gated by the browsers the project targets —
`.browserslistrc` or the `browserslist` key in `package.json`; with neither, the
build tool's default target.

### 4. Guardrails — consistency beats novelty

- The project's `CLAUDE.md` and the sibling code's conventions win over a newer
  idiom. Do not add a second pattern next to an established one — a `#[Scope]`
  method in a model whose siblings use `scopeFoo()`, `casts()` next to `$casts` —
  unless the task is a refactor; then migrate consistently, in one change.
- **No drive-by rewrites.** Code the task does not touch stays as it is, however
  dated. Name the opportunity in the summary instead ("`Order` still uses
  `$casts`; the installed 12.x supports `casts()`").
- Never introduce an API the installed version deprecates, and never one newer
  than the installed version, the PHP floor, or (for JS/CSS) the browserslist
  target — it does not run.
- No new dependency for a capability the framework already ships (`once()`
  instead of a memoization package, `Number` instead of a formatting package,
  `Http::retry()` instead of a retry library).

## Proving it with tests

`reference/testing-patterns.md` → *New screen: reachability & authorization*
has the Pest patterns: the link is rendered for an allowed user and not for a
denied one, the route answers 403 for the denied user and for another tenant's
record, and — where the project already has a browser-test stack — one test
that reaches the screen by clicking instead of visiting its URL.

## Pre-finish self-check

Run this before calling the change done. Answer each line for the diff, not for
the whole app; skip a dimension only when its "applies when" does not hold.

- **reachability** — Every new screen has a menu entry and/or an inbound link
  from its natural parent, added in this change. Link visibility uses the same
  ability as the route. Renamed/removed screens left no dead link (old name and
  path grepped). URL-only pages are named as such.
- **security** — Every new route/action has the neighbours' middleware plus an
  object-scoped policy check. Input goes through a FormRequest and
  `validated()`. No scope bypass, no interpolated SQL/shell, no 500 on crafted
  input, no secret in the diff.
- **performance** — No per-row query (eager loads match what the view touches).
  Lists paginate; batches chunk. New FKs and filter/sort columns are indexed, no
  duplicate index. Slow work is queued.
- **ui_ux** — Matches sibling screens. Every string is `__()` with an English
  key in the module lang file and resolves to text. Disabled controls say why;
  controls have names/labels; empty/loading/error states exist; destructive
  actions confirm.
- **framework** — Versions were read from `composer.lock` / `composer.json`, not
  assumed. New and changed code uses nothing newer than the installed versions,
  the PHP floor, and (for JS/CSS) the browserslist target, and nothing they
  deprecate. Nothing is hand-rolled that the
  installed framework ships. No second pattern next to an established one.
  Untouched code was left alone, and its modernization opportunities are named in
  the summary. The docs page is named where a choice was not obvious.

## Reporting findings (reviewers)

When reviewing rather than building, tag each finding so it lines up with the
QA report of `teamwork-task-test`:

- `dimension` — one of `ui_ux`, `performance`, `security`, `reachability`,
  `framework` (or a non-cross-cutting area such as style or architecture);
- `severity` — `high` / `medium` / `low` (Critical / Important / Nice-to-have);
  `info` for `framework` recommendations;
- `location` — `path/to/file.php:41`; a finding you cannot point at a line for
  is not a finding;
- `in_this_diff` — whether this change introduced it or only walked past it.

Typical reachability findings: a new route/screen with no menu entry and no
inbound link; a dead link after a rename; a menu item visible to a role the
route refuses (or hidden from a role the route admits).

**`framework` is advisory.** A reviewer recommends; it does not rewrite working
code, and a recommendation never blocks approval or counts as a defect:

- Tag it `severity: info` (`advisory: true`) and list it in its own short
  section after the findings, phrased as an optional suggestion for a later
  refactor.
- Only lines of the diff. Each recommendation points at `file:line`, names the
  installed version and the docs page, and gives the concrete replacement that
  version offers. "Could be more modern" without such a replacement is not a
  recommendation. At most five per review — say how many you dropped.
- A deprecated API that already emits deprecation warnings is an `info`
  recommendation too.
- **One exception is a real finding, not a recommendation:** an API newer than
  the installed version or the PHP floor, or one the installed version has
  removed, does not run — a parse or fatal error, or silently ignored (a
  `casts()` method on Laravel 10). Report it as `high` (Critical) under the
  dimension it breaks, or as a plain correctness defect.
