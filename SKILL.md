---
name: prod-guard
description: Enforce 7 production safety guarantees — regression tests, idempotent writes, database constraints and indexes, circuit breakers and fallbacks, RBAC with IDOR prevention and input sanitization, structured logs with trace IDs, and reversible zero-downtime migrations. Use when building or reviewing APIs, auth and permissions, multi-tenant data, database schemas and migrations, payments or other critical writes, background jobs, external integrations, or deploys. Companion to prod-ready.
---

# Prod-Guard: Production Safety & Reliability Guarantees

*Author: Flávio Fontes*

This skill enforces code that is **correct, secure and recoverable** in production, not just fast. It is the companion of `prod-ready` (performance and design system): `prod-ready` makes an app fast and well designed; `prod-guard` makes sure it does not leak data, duplicate writes, corrupt the database, cascade failures or break during deploys.

Whenever an API, data model, migration, job, integration or deploy is planned, written or refactored under this skill, the 7 guarantees below apply. Full rules, anti-patterns and examples (SQL, PHP/Laravel, TypeScript/Node/Prisma) are in the reference files.

---

## 🎯 Activation Triggers

Apply this skill when:
- The user asks for `prod-guard`, `/prod-guard`, "make this secure", "production-safe", "harden this", "review for security".
- Code touches authentication, authorization, roles, permissions, tenants or user-owned records.
- Code creates or changes a database schema, migration, constraint or index.
- Code performs a write that must not happen twice: payments, orders, emails, webhooks, stock, balances.
- Code calls an external API, runs background jobs, or is about to be deployed.
- A bug is being fixed (a regression test is required).

---

## ⚖️ Precedence Rule

When two rules conflict, the higher one wins:

1. **Security** (G5) — never trade it for speed or convenience.
2. **Data integrity** (G2, G3, G7) — the database must never end up wrong.
3. **Reliability** (G1, G4, G6) — failures are contained, visible and testable.
4. **Performance** (`prod-ready` rules 1–25).
5. **Aesthetics** (`prod-ready` design system).

Consequences: a cache never bypasses authorization; an optimistic UI never replaces server-side validation; a retry never runs on a non-idempotent write; client-side request deduplication never applies to writes.

---

## 📏 Applicability Levels

Each guarantee carries a level, so it is applied where it matters and not as ceremony:

- **Always** — every feature, no exceptions.
- **On writes / state changes** — whenever data is created, changed or deleted.
- **On external dependencies** — whenever the code calls something it does not control.

---

## 🛡️ The 7 Production Guarantees

### Security — `references/security-rules.md`

**G5. Authorization, IDOR prevention & input sanitization** *(Always)*
Every endpoint authenticates the caller and authorizes the action **on the specific record** (ownership or tenant), on the server, by default deny. Tenant scope is applied in the query itself, never after fetching. All input is validated against an explicit schema (type, length, format, allow-list); output is encoded for its context; SQL is always parameterized. Secrets never live in code or logs.

### Data integrity — `references/data-integrity-rules.md`

**G2. Idempotent critical operations** *(On writes)*
Any write that can be retried — by the user, the client, a queue or a webhook sender — produces the same result when repeated. Use an `Idempotency-Key` stored with a unique constraint, natural unique keys, or upserts. Jobs are delivered at-least-once, so every job handler is idempotent.

**G3. Database integrity: constraints & indexes** *(Always)*
The database, not only the application, enforces the invariants: `NOT NULL`, `UNIQUE`, `FOREIGN KEY` with an explicit `ON DELETE`, `CHECK`, correct types for money and time. Every foreign key, every filter column and every cursor ordering has a matching index. Multi-step writes run inside a transaction.

**G7. Reversible migrations & zero-downtime deployment** *(On writes, for schema changes)*
Every migration has a working `down`. Breaking schema changes follow **expand → migrate → contract** across separate deploys, so old and new code run against the same schema. No long table locks; large backfills run in batches. Deploys are rolling, with graceful shutdown and health checks.

### Reliability — `references/reliability-rules.md`

**G1. Automated regression tests** *(Always for business logic)*
Every bug fix ships with a test that fails before the fix and passes after it. Critical paths (auth, permissions, money, data writes) have tests that run in CI and block the merge when they fail. Tests assert behavior, not implementation details.

**G4. Resilience & graceful degradation** *(On external dependencies)*
Every external call has a timeout, a circuit breaker (closed → open → half-open), retries **only on idempotent operations** with exponential backoff and jitter, and an explicit fallback (stale cache, partial response, or a clear "temporarily unavailable"). A failing optional dependency never takes down the main flow.

**G6. Observability: structured logs & trace IDs** *(Always)*
Logs are structured JSON with a level, a message and context. Every request gets a trace ID (accepted from `traceparent`/`X-Request-Id` or generated), returned in the response and propagated to downstream calls and queued jobs. No secrets or personal data in logs. Errors are reported with enough context to reproduce them.

---

## 🔗 Relationship with prod-ready

`prod-guard` adds to `prod-ready`; it does not repeat it. Where both touch the same topic:

| prod-ready rule | prod-guard adds |
|---|---|
| 4. Request deduplication | Deduplication is never idempotency: writes use G2 |
| 6. Pagination | Cursor order needs a unique tie-breaker and an index (G3); tenant filter before pagination (G5) |
| 8. Background queues | Jobs are idempotent (G2), retried with backoff and dead-lettered (G4), and carry the trace ID (G6) |
| 11. Redis cache | Cache keys include tenant/user scope (G5); Redis down → read from the database (G4) |
| 13. Timeouts | Circuit breaker, retries and fallbacks are defined in G4 |
| 15. Optimistic UI | Retries send an `Idempotency-Key` (G2); the server still authorizes (G5) |
| 22. Cache headers | Authenticated responses use `private, no-store` (G5) |
| 23. Stateless | Graceful shutdown and readiness checks for zero-downtime deploys (G7) |
| 25. RUM & slow-network tests | Part of the CI gate together with regression tests (G1) |

---

## 🛠️ Execution Checklist

Before delivering code, check and reflect in the result:
- [ ] Does every endpoint authenticate the caller and check permission **on this record**, scoped by tenant in the query? (G5)
- [ ] Is every input validated by an explicit schema, and every SQL query parameterized? (G5)
- [ ] Can this write be repeated safely? Is there an idempotency key or unique constraint? (G2)
- [ ] Are invariants enforced by database constraints, and do foreign keys and filters have indexes? (G3)
- [ ] Does the migration have a working `down`, and is a breaking change split into expand/contract? (G7)
- [ ] Does every external call have a timeout, a circuit breaker and a defined fallback? Are retries limited to idempotent operations? (G4)
- [ ] Are logs structured, free of secrets, and tagged with the trace ID? (G6)
- [ ] Does the bug fix or critical path ship with a test that would catch a regression? (G1)
