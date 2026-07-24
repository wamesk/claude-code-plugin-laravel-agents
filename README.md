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
| **laravel-backend** | Controllers, services, FormRequests, API Resources, file uploads, auth. Follows the service-layer pattern. |
| **sql-database** | Migrations (ULID, indexes, enums, `sort`, `dateTimeTz`), Eloquent query optimization (N+1), model configuration. |
| **api-architect** | RESTful API design, versioning (v1, v2…), rate limiting, middleware, FormRequests, API Resources/Collections, Scramble docs. |
| **pest-tester** | Pest feature tests, helpers, factories, reusable scenarios. Nova/Dusk browser tests when `laravel-nova-agents` is installed. |
| **code-reviewer-laravel** | Code review against WAME standards — Pint, type safety, response format, service layer, N+1, translations, security. |
| **security-auditor** | OWASP Top 10 and Laravel-specific security audits with remediation guidance. |
| **performance-optimizer** | Query/caching/response-time optimization with before/after metrics. |

## Skills

| Skill | What it holds |
|-------|---------------|
| **wame-laravel-standards** | API response format (with `BaseCollection` pagination), database conventions, service layer, testing patterns. The agents defer here for examples. |
| **wame-package-development** | How to create a clean composer package/module per [laravelpackage.com](https://www.laravelpackage.com): stubs (ServiceProvider, `composer.json`, migration, routes, view, test) + host-app `composer.json` path-repository wiring so local modules are discovered. |
| **wame-security-checklist** | OWASP Top 10 + Laravel security reference used by `security-auditor`. |
| **wame-performance-playbook** | Query optimization, caching strategies, file-storage/CDN reference used by `performance-optimizer`. |

## Design

- **Thin agents, deep skills.** Each agent is ~150 lines of persona, boundaries, and decision
  rules; the reference material lives in `skills/*/reference/` and loads only when relevant
  (progressive disclosure). Agents invoke skills by name via the `Skill` tool.
- **Standard, not project-specific.** Namespaces in examples use a `Vendor\Module` placeholder;
  the real namespace is defined per-project in `CLAUDE.md`.
- **Data via migrations, not seeders.** Reference data and defaults ship as idempotent `*_seed_*`
  migrations; factories remain for tests.

## License

MIT
