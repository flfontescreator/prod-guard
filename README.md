# 🛡️ Prod-Guard — Production Safety & Reliability Skill

> **7 production guarantees that stop insecure, fragile code before it ships: no data leaks between customers, no duplicated payments, no corrupted database, no cascading failures, no broken deploys.**

By **Flávio Fontes** · Companion skill to [`prod-ready`](https://github.com/flfontescreator/prod-ready)

---

## The problem

Code can be fast, clean and well designed, and still fail in production in ways that cost money and trust:

- ❌ **Data leaks (IDOR)** — change the id in the URL and see another customer's invoice.
- ❌ **Duplicated writes** — a network retry charges the card twice or creates two orders.
- ❌ **Silent data corruption** — no constraints, so the database accepts orphans, duplicates and negative stock.
- ❌ **Cascading failures** — one slow payment or email provider freezes the whole app.
- ❌ **Blind debugging** — free-text logs, no trace ID, no way to follow what happened to a request.
- ❌ **Broken deploys** — a column rename takes the app down, and the migration cannot be rolled back.
- ❌ **Bugs that come back** — fixes without tests, so the same bug returns two releases later.

`prod-guard` turns each of these into an explicit rule, applied whenever the code touches auth, data, money, integrations or deploys.

---

## ✨ The 7 Production Guarantees

| # | Guarantee | What it enforces |
|---|---|---|
| **G1** | **Automated regression tests** | Every bug fix ships with a test that failed before the fix. Auth, permissions, money, idempotency and migrations are always tested. Tests block the merge in CI. |
| **G2** | **Idempotent critical operations** | `Idempotency-Key` for retried writes, natural unique keys and upserts, idempotent jobs and webhooks, jobs dispatched after commit (outbox). |
| **G3** | **Database integrity** | `NOT NULL`, `UNIQUE`, foreign keys with explicit `ON DELETE`, `CHECK`; correct types for money and time; indexes for foreign keys, filters and cursor ordering; transactions and locking. |
| **G4** | **Resilience & graceful degradation** | Timeouts everywhere, retries only on idempotent operations with backoff and jitter, circuit breakers, explicit and honest fallbacks, bulkheads and dead-letter queues. |
| **G5** | **Security, RBAC & IDOR prevention** | Deny by default, authorization on the specific record, tenant scope in every query, schema validation, parameterized SQL, output encoding, no mass assignment, secrets out of code and logs, rate limits. |
| **G6** | **Observability** | Structured JSON logs, a trace ID on every request propagated to calls and jobs, no secrets or personal data in logs, error tracking, RED metrics and symptom-based alerts. |
| **G7** | **Reversible migrations & zero-downtime deploys** | A tested `down` for every migration, expand → migrate → contract for breaking changes, online DDL, batched backfills, rolling deploys with graceful shutdown. |

A **precedence rule** resolves conflicts: security → data integrity → reliability → performance → aesthetics. A cache never bypasses authorization; a retry never repeats a non-idempotent write.

Every guarantee has an **applicability level** (*Always*, *On writes*, *On external dependencies*), so it is applied where it matters instead of adding ceremony to every line of code.

---

## 📚 What's inside

- **`SKILL.md`** — activation triggers, precedence rule, the 7 guarantees in short form, how it relates to `prod-ready`, and an execution checklist.
- **`references/security-rules.md`** — G5 in detail: 9 rules with anti-patterns, patterns and verification.
- **`references/data-integrity-rules.md`** — G2, G3 and G7: 12 rules covering idempotency, constraints, indexes, transactions, migrations and deploys.
- **`references/reliability-rules.md`** — G1, G4 and G6: 13 rules covering tests, resilience and observability.

Code examples in **SQL** (PostgreSQL and MySQL), **PHP / Laravel** and **TypeScript / Node / Prisma**.

---

## 🤝 Works with prod-ready

`prod-ready` makes an app **fast and well designed** (25 performance rules and a design system). `prod-guard` makes it **safe and recoverable**. They do not repeat each other: where both touch the same topic, `prod-guard` adds the safety side.

| prod-ready rule | prod-guard adds |
|---|---|
| 4. Request deduplication | Deduplication is never idempotency: writes use G2 |
| 6. Pagination | Unique cursor tie-breaker and index (G3); tenant filter first (G5) |
| 8. Background queues | Idempotent jobs, backoff, dead-letter queue, trace ID (G2, G4, G6) |
| 11. Redis cache | Tenant-scoped keys (G5); database fallback when Redis is down (G4) |
| 13. Timeouts | Circuit breaker, retries and fallbacks (G4) |
| 15. Optimistic UI | `Idempotency-Key` on retries; server still authorizes (G2, G5) |

Using both is recommended. `prod-guard` also works on its own, for backend-only projects.

---

## 🚀 Installation

Unzip the package into your skills folder.

**All projects:**

```bash
unzip prod-guard.zip -d ~/.claude/skills/
```

**One project only** (from the project root):

```bash
mkdir -p .claude/skills
unzip prod-guard.zip -d .claude/skills/
```

The folder must end up as `skills/prod-guard/SKILL.md`.

---

## 💡 Usage

Invoke it explicitly, or let it activate when the work touches auth, data, money, integrations or deploys:

```text
> Build the orders API with prod-guard
```

```text
> Review this endpoint for IDOR and missing authorization
```

```text
> Rename the column `state` to `status` without downtime
```

```text
> Make the payment webhook idempotent and add a regression test
```

---

## 📁 Structure

```text
prod-guard/
├── SKILL.md                          # Triggers, precedence, the 7 guarantees, checklist
├── README.md                         # This file
├── LICENSE                           # License terms
└── references/
    ├── security-rules.md             # G5
    ├── data-integrity-rules.md       # G2, G3, G7
    └── reliability-rules.md          # G1, G4, G6
```

---

## 📄 License

Proprietary. © 2026 Flávio Fontes. See [LICENSE](LICENSE) for the terms of use.
