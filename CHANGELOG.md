# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
