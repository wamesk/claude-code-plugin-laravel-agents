# laravel-agents

Specialized Laravel subagents and companion skills for Claude Code, by [WAME](https://wame.sk).

Seven thin agent personas (Nova-free) that defer to progressively-disclosed skills instead of
carrying thousands of lines of examples in their prompt. Part of the `wame` marketplace.

## Installation

```
/plugin marketplace add wamesk/claude-code
/plugin install laravel-agents@wame
```

For Laravel Nova projects, also install [`laravel-nova-agents`](https://github.com/wamesk/claude-code-plugin-laravel-nova-agents) — the `pest-tester` agent will then pick up Nova/Dusk browser-testing patterns automatically.

## Agents

| Agent | Use it for |
|-------|-----------|
| **laravel-backend** | Controllers, services, FormRequests, API Resources, file uploads, auth. Follows the service-layer pattern and applies the five cross-cutting quality rules while building — a new screen ships with its menu entry and inbound links, and new code uses the idioms of the Laravel/PHP/Pest versions the project has installed. |
| **sql-database** | Migrations (ULID, indexes, enums, `sort`, `dateTimeTz`), Eloquent query optimization (N+1), model configuration. |
| **api-architect** | RESTful API design, versioning (v1, v2…), rate limiting, middleware, FormRequests, API Resources/Collections, Scramble docs. |
| **pest-tester** | Pest feature tests, helpers, factories, reusable scenarios, and reachability + authorization tests for every new screen (one click-through test where a browser stack already exists). Nova/Dusk browser tests when `laravel-nova-agents` is installed. |
| **code-reviewer-laravel** | Code review against WAME standards — Pint, type safety, response format, service layer, N+1, translations, security — plus the four cross-cutting dimensions (`ui_ux`, `performance`, `security`, `reachability`) tagged with the same keys as `teamwork-task-test`, and advisory `framework` recommendations (an API newer than the installed version is a real finding). |
| **security-auditor** | OWASP Top 10 and Laravel-specific security audits with remediation guidance. |
| **performance-optimizer** | Query/caching/response-time optimization with before/after metrics. |

## Skills

| Skill | What it holds |
|-------|---------------|
| **wame-laravel-standards** | API response format (with `BaseCollection` pagination), database conventions, service layer, testing patterns, and the cross-cutting quality reference (UI/UX, performance, security, page reachability, framework idioms of the installed versions + a pre-finish self-check). The agents defer here for examples. |
| **wame-package-development** | How to create a clean composer package/module per [laravelpackage.com](https://www.laravelpackage.com): stubs (ServiceProvider, `composer.json`, migration, routes, view, test) + host-app `composer.json` path-repository wiring so local modules are discovered. |
| **wame-security-checklist** | OWASP Top 10 + Laravel security reference used by `security-auditor`. |
| **wame-performance-playbook** | Query optimization, caching strategies, file-storage/CDN reference used by `performance-optimizer`. |

## Design

- **Thin agents, deep skills.** Each agent is under 100 lines of persona, boundaries, and decision
  rules; the reference material lives in `skills/*/reference/` and loads only when relevant
  (progressive disclosure). Agents invoke skills by name via the `Skill` tool.
- **Standard, not project-specific.** Namespaces in examples use a `Vendor\Module` placeholder;
  the real namespace is defined per-project in `CLAUDE.md`.
- **Data via migrations, not seeders.** Reference data and defaults ship as idempotent `*_seed_*`
  migrations; factories remain for tests.
- **Quality built in, not bolted on.** The agents that build, test, and review code share one
  reference — `skills/wame-laravel-standards/reference/cross-cutting-quality.md` — see below.

## Cross-cutting quality (since 1.1.0)

The [`teamwork-task-test`](https://github.com/wamesk/claude-code-plugin-teamwork-task-test) plugin
reviews every task's diff for four things the acceptance criteria rarely mention, plus a fifth,
advisory one — `framework`. This plugin makes the developer apply all five **while building**, under
the same keys:

| Key | Build-time rule (short) |
|-----|-------------------------|
| `reachability` | Every new screen is reachable by clicking, in the same change: a menu entry for the roles that use it and/or inbound links from related screens (Blade / Livewire / Inertia / SPA). Link visibility uses the same ability as the route. Renames leave no dead links; URL-only pages are named as such. |
| `security` | Neighbours' middleware plus an object-scoped policy check on every new route/action, FormRequest + `validated()`, no tenant-scope bypass, no 500 on crafted input. Hiding a menu item is not access control. |
| `performance` | Eager-load what each row touches, paginate lists, `chunkById` batches, index new foreign keys and filter/sort columns, queue slow work. |
| `ui_ux` | Follow sibling screens, translate every string with an English key in the module lang file, design loading/empty/error states, disabled controls say why, destructive actions confirm. Applies only when the change renders something. |
| `framework` | Read the installed versions from `composer.lock` and the PHP floor from `composer.json` first; look non-obvious APIs up in that version's docs (Laravel Boost `search-docs` → context7 → official docs); use the version's idioms and built-in features in new and changed code (PHP 8.1–8.4, Laravel 10–13, Pest 2–4, each tagged with its minimum version); never an API newer than installed or deprecated; the project's `CLAUDE.md` and sibling conventions win; no drive-by rewrites of untouched code. |

`laravel-backend` runs the pre-finish self-check, `pest-tester` proves reachability and
authorization with tests, and `code-reviewer-laravel` reports findings tagged with the same keys.
`framework` is advisory on the review side, exactly as in `teamwork-task-test`: `code-reviewer-laravel`
lists `info` recommendations in their own section and never blocks on them or rewrites working code —
except an API newer than the installed version (or removed by it), which does not run and is a real
finding. `laravel-backend` and `code-reviewer-laravel` may call Laravel Boost (`application-info`,
`search-docs`), context7, and WebFetch for the docs lookup; the other agents carry a short pointer to
the rule.
Nova screens (menu, relation tabs, lenses, dashboards) are covered by `laravel-nova-agents`.

## License

MIT
