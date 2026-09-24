# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-09-24

Aligns the build side with the QA side: `teamwork-task-test` 1.1.0 reviews every
task's diff for UI/UX, performance, security, and page reachability, and until
now nothing on the build side asked the developer to think about them before QA
did. Most of all, new screens kept shipping without a menu entry or a link from
related pages — findable only by typing the URL.

The same release adds a fifth dimension, `framework`: use the idioms and built-in
features of the framework and language versions the project actually has
installed. The agents wrote from model memory, which is stale both ways — dated
patterns on a new install (a `$casts` property on Laravel 12), and features an
older install does not have, which is a bug rather than style (a property hook on
PHP 8.3 is a parse error; a `casts()` method on Laravel 10 is never called, so
the attributes are silently not cast). `teamwork-task-test` 1.2.0 reviews it as
an advisory dimension; here it is a build rule for new and changed code and an
advisory review step.

### Added

- `skills/wame-laravel-standards/reference/cross-cutting-quality.md` — the four
  dimensions as build-time rules under the same keys QA uses (`ui_ux`,
  `performance`, `security`, `reachability`), each with an "applies when" so a
  pure backend change does not grow UI boilerplate, a pre-finish self-check, and
  the finding format for reviewers (`dimension` / `severity` / `location` /
  `in_this_diff`). Reachability is kept generic — Blade, Livewire, Inertia, SPA
  navigation, `route:list` vs the rendered menu, breadcrumbs — and points to
  `laravel-nova-agents` for Nova.
- `cross-cutting-quality.md` → `framework`: detect the installed versions once
  per task (zsh-safe `jq` over `composer.lock`, the PHP floor from
  `composer.json` `require.php` / `config.platform.php`, `package.json` and the
  browserslist target for views; Laravel Boost `application-info`, whose `php_version` is the MCP
  server's PHP, not the floor); look them up in current docs in the order
  Boost `search-docs` → context7 → official docs; prefer the installed
  version's idioms, with PHP 8.1–8.4, Laravel 10–13, and Pest 2–4 examples each
  tagged with the minimum version that has them (checked against the framework
  sources and changelogs, not recalled — e.g. `whereAll` 10.47, `once()` /
  `casts()` 11.0, `defer()` / `Cache::flexible()` 11.23, `#[UseFactory]` 11.39,
  `#[Scope]` 12.4, `#[UsePolicy]` 12.18, model attributes 13.0, arch presets
  Pest 3); guardrails (the project's `CLAUDE.md` and sibling conventions win, no
  second pattern next to an established one, no drive-by rewrites of untouched
  code, nothing deprecated or newer than installed, no new dependency for a
  built-in capability); a `framework` line in the pre-finish self-check; and the
  reviewer rules — `framework` is advisory (`severity: info`, `advisory: true`,
  own section, at most five, never blocks), except an API newer than the
  installed version or removed by it, which is a real finding.
- `reference/testing-patterns.md` → "New screen: reachability & authorization":
  Pest tests proving the menu link is shown to allowed users and hidden from
  denied ones, 403 for denied users and foreign-tenant records, an Inertia
  shared-prop variant (the HTML check proves nothing for a client-rendered menu),
  and one click-through test where a browser stack already exists.

### Changed

- `laravel-backend` applies the four rules while building: a new web route or
  screen ships with its menu entry and inbound links in the same change; policy +
  FormRequest on every new entry point; eager loading, pagination, and indexes on
  new lists; translated strings and designed states when the change renders UI.
  It runs the self-check before declaring the change done. It also applies
  `framework` to new and changed code: versions read from `composer.lock` first,
  the installed version's idioms looked up rather than recalled, nothing newer or
  deprecated, untouched code left alone with the opportunity named in the
  summary.
- `pest-tester` covers every new screen with authorization tests (denied user,
  foreign tenant) and a reachability test, and never proves reachability by
  visiting the URL. The blanket "skip browser testing without Nova" rule now
  allows the single click-through test when the project already has a browser
  stack.
- `code-reviewer-laravel` gains review step 9 for the four dimensions with the
  same keys as `teamwork-task-test`: a new route/screen without a menu entry or
  inbound link, a dead link after a rename, and menu visibility that disagrees
  with route authorization are findings. Severities map Critical / Important /
  Nice-to-have → high / medium / low. Review step 10 adds `framework` as advisory
  `info` recommendations in their own section (line, installed version,
  concrete replacement, docs page; at most five), never a reason to block or to
  rewrite working code — except an API newer than the installed version or
  removed by it, which is a Critical finding because it does not run.
- `laravel-backend` and `code-reviewer-laravel` get `WebFetch`, Laravel Boost
  `application-info` / `search-docs`, and context7 `resolve-library-id` /
  `query-docs` in their `tools`, so the docs lookup the `framework` rule asks for
  is actually callable from the subagent.
- `api-architect`, `sql-database`, `pest-tester`, `performance-optimizer`, and
  `security-auditor` carry a one-line `framework` pointer where it is natural —
  e.g. `sql-database`: `casts()` needs Laravel 11; `pest-tester`: arch presets
  need Pest 3 and `visit()` Pest 4, whatever version the skill's examples target;
  `security-auditor`: a remediation that calls a newer API does not run. The
  `wame-performance-playbook` "Supported versions" block (Laravel 13.x) now says
  it is the examples' target and the installed version decides.
- `security-auditor` treats hidden-but-open screens as an A01 finding — hiding a
  menu item is not access control.
- `wame-laravel-standards` SKILL.md links the new reference and adds the
  reachability and framework non-negotiables. Its "Supported versions" block now
  says those are the versions the examples target, not a licence — the installed
  versions decide.

### Fixed

- `wame-package-development` → `reference/composer-wiring.md` and step 4 of
  its `SKILL.md`: the install command `composer require wamesk/example-widget:*`
  left the `*` unquoted.
  Claude Code runs snippets in zsh on macOS, where an unmatched glob is a hard
  error — the command aborted with `no matches found` and composer never ran.
  The constraint is now quoted: `composer require 'wamesk/example-widget:*'`.
- `wame-laravel-standards` SKILL.md claimed attribute-based observer/factory
  registration "requires Laravel 11+". `#[ObservedBy]` exists since 10.44 and
  `#[UseFactory]` only since 11.39, so on Laravel 11.0–11.38 the attribute is
  silently ignored and the factory is resolved by naming convention instead —
  wrong for a module whose factories live outside `Database\Factories`. It now
  names both minimum versions and the fallback (`Model::observe()` in a
  provider, `newFactory()`).

## [1.0.0] - 2026-07-24

### Added

- Seven Laravel subagents: `laravel-backend`, `sql-database`, `api-architect`,
  `pest-tester`, `code-reviewer-laravel`, `security-auditor`, `performance-optimizer`.
  Each is a thin persona (~150 lines) that defers to companion skills instead of
  inlining reference material.
- Skill `wame-laravel-standards` — API response format, database conventions
  (ULID, enums, `sort`, `dateTimeTz`, data via migrations not seeders), service
  layer, and testing patterns, split across `reference/` for progressive disclosure.
- Skill `wame-package-development` — how to create a clean composer package/module
  following [laravelpackage.com](https://www.laravelpackage.com), with adapted
  stubs and `composer.json` path-repository wiring so the host app discovers local
  modules.
- Skill `wame-security-checklist` — OWASP Top 10 and Laravel-specific security
  audit reference.
- Skill `wame-performance-playbook` — query optimization, caching strategies, and
  file-storage/CDN reference.
- Migrated from a previous one-way `.claude` copy script into the `wame`
  marketplace so agents version and update like any other plugin.
