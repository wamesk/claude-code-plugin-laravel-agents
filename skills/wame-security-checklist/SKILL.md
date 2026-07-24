---
name: wame-security-checklist
description: "Use when auditing or hardening a Laravel application for security — reviewing code for vulnerabilities, checking auth/authorization, or preparing an app for production. Covers the OWASP Top 10 (broken access control/IDOR, SQL injection, XSS, CSRF, SSRF, cryptographic failures, misconfiguration), Sanctum token auth, authorization/ownership checks, mass assignment, file-upload security, rate limiting, CORS, security headers, secrets/env config, security logging, plus a pre-production checklist and audit report template. Trigger phrases: 'security audit', 'security review', 'is this secure', 'OWASP', 'SQL injection', 'IDOR', 'harden before production', 'check for vulnerabilities', 'auth/authorization review'."
---

# WAME Security Checklist

Reference material for auditing and hardening Laravel applications. It maps the
OWASP Top 10 onto concrete Laravel checks with vulnerable-vs-secure code, and
adds the framework-specific concerns that don't fit a single OWASP category.

The bulk lives in `reference/`; load only the file you need.

## When to use

- Running a full security audit of a Laravel app or module.
- Reviewing a controller, route, model or upload handler for vulnerabilities.
- Hardening authentication/authorization (Sanctum, policies, ownership).
- Preparing an app for production (env, headers, CORS, secrets, dependencies).
- Producing a structured audit report with severities and remediation.

## Supported versions

These checks target the current supported major releases; pin exact constraints
in the project `composer.json` and treat any end-of-life release as an A06
finding:

- **Laravel:** current supported major version (`laravel/framework`).
- **PHP:** the version required by that Laravel release.
- **Laravel Sanctum:** the version matching that Laravel release.
- **Pest:** current supported major version (dev-only).

The `Illuminate\Support\Facades\Process` facade used in the injection examples is
available from the supported Laravel version.

## Reference map

- **`reference/owasp-top10.md`** — OWASP Top 10 (A01-A10) with secure examples:
  broken access control & path traversal (A01), cryptographic failures (A02),
  SQL/command injection (A03), insecure design + password policy + reusable
  SecurityHeaders middleware (A04), misconfiguration + reusable
  SecureExceptionHandler (A05), outdated components (A06), auth failures summary
  (A07), data-integrity/deserialization (A08), logging & monitoring (A09), SSRF
  (A10).
- **`reference/laravel-security.md`** — mass assignment, file-upload security,
  CSRF/XSS, rate limiting, CORS, secrets & environment configuration, the
  pre-production audit checklist, and the audit report template.
- **`reference/api-auth-sanctum.md`** — Sanctum token lifecycle & expiration,
  login rate-limit/lockout, session security, authorization & ownership checks
  (IDOR prevention), and security testing commands.

## How to apply

1. Prioritize by severity: Critical (SQLi, auth bypass, RCE, IDOR on sensitive
   data) first, then High, Medium, Low.
2. For each finding, show the vulnerable code, the exploit/impact, and a concrete
   remediation — use the report template in `laravel-security.md`.
3. Namespaces in examples use a `Vendor\Module` placeholder; the real namespace
   is defined per-project in `CLAUDE.md`.
