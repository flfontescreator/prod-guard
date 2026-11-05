# Reliability Rules — G1 Regression Tests, G4 Resilience, G6 Observability

**Precedence:** below security and data integrity, above performance. A fast system that fails silently, cascades failures or breaks again with every fix is not production-ready.

---

## Index

**G1 — Automated regression tests** *(Always for business logic)*
1. [Every bug fix ships with a regression test](#g1-1-every-bug-fix-ships-with-a-regression-test)
2. [Critical paths always have tests](#g1-2-critical-paths-always-have-tests)
3. [The right test at the right level](#g1-3-the-right-test-at-the-right-level)
4. [Tests gate the merge, and stay deterministic](#g1-4-tests-gate-the-merge-and-stay-deterministic)

**G4 — Resilience & graceful degradation** *(On external dependencies)*
5. [Timeouts on every network call](#g4-5-timeouts-on-every-network-call)
6. [Retries only on idempotent operations, with backoff and jitter](#g4-6-retries-only-on-idempotent-operations-with-backoff-and-jitter)
7. [Circuit breaker: closed, open, half-open](#g4-7-circuit-breaker-closed-open-half-open)
8. [An explicit fallback for every dependency](#g4-8-an-explicit-fallback-for-every-dependency)
9. [Bulkheads and dead-letter queues](#g4-9-bulkheads-and-dead-letter-queues)

**G6 — Observability** *(Always)*
10. [Structured JSON logs](#g6-10-structured-json-logs)
11. [A trace ID on every request, propagated everywhere](#g6-11-a-trace-id-on-every-request-propagated-everywhere)
12. [No secrets or personal data in logs](#g6-12-no-secrets-or-personal-data-in-logs)
13. [Error tracking, metrics and alerts](#g6-13-error-tracking-metrics-and-alerts)

---

## G1 — Automated Regression Tests

### G1-1. Every bug fix ships with a regression test

- **Rule**: Before fixing a bug, write a test that reproduces it and **fails**. Fix the code until the test **passes**. The test stays in the suite forever, so the same bug cannot return unnoticed.
- **Anti-pattern**: fixing the line, testing it by hand once, and moving on.
- **Pattern** — a fixed IDOR bug, locked by a test:

```php
// Laravel (PHPUnit) — tests/Feature/InvoiceAccessTest.php
public function test_user_cannot_view_an_invoice_from_another_tenant(): void
{
    $invoice  = Invoice::factory()->create();          // belongs to tenant A
    $intruder = User::factory()->create();             // belongs to a different tenant

    $this->actingAs($intruder)
        ->getJson("/api/invoices/{$invoice->id}")
        ->assertNotFound();
}
```

```ts
// TypeScript (Vitest) — a fixed double-charge bug
// `api`, `body` and `tenantId` are your test helpers and fixtures.
it('does not create a second order when the same Idempotency-Key is retried', async () => {
  const key = crypto.randomUUID();

  const first = await api.post('/api/orders', body, { 'Idempotency-Key': key });
  const retry = await api.post('/api/orders', body, { 'Idempotency-Key': key });

  expect(retry.status).toBe(first.status);
  expect(await prisma.order.count({ where: { tenantId } })).toBe(1);
});
```

- **Name the test after the behavior**, not the ticket: `user_cannot_view_an_invoice_from_another_tenant` explains itself; `test_bug_1432` does not.

### G1-2. Critical paths always have tests

- **Rule**: These areas have automated tests even when there was no bug yet:
  - **Authentication and authorization** — unauthenticated → `401`; other tenant → `404`; missing permission → `403`.
  - **Money and quantities** — totals, rounding, discounts, refunds, stock.
  - **Idempotency** — the same request twice produces one effect (G2).
  - **Constraints** — the database rejects invalid data (duplicate, orphan, negative quantity) (G3).
  - **Migrations** — `up`, `down`, `up` again runs clean (G7).
  - **Input validation** — wrong type, oversized value and unknown field → `422` (G5).
- **Performance regressions** that matter can be tested too, for example the number of queries of a list endpoint (catches N+1, prod-ready rule 9):

```php
// Laravel: fail if the list endpoint starts issuing one query per row
DB::enableQueryLog();
$this->actingAs($user)->getJson('/api/orders')->assertOk();
$this->assertLessThanOrEqual(5, count(DB::getQueryLog()));
```

### G1-3. The right test at the right level

- **Unit tests** for pure logic: calculations, parsing, state transitions. Fast, no I/O.
- **Integration tests** for anything that touches the database: queries, constraints, transactions, policies. Run them against the **same database engine** as production (a PostgreSQL or MySQL container), not an in-memory substitute with different behavior, and not mocks of the ORM.
- **End-to-end tests** (Playwright, Cypress, Laravel Dusk) for a few critical user journeys only: sign in, the main purchase or submission flow. They are slow and fragile, so keep them few.
- **Test behavior, not implementation**: assert on HTTP responses, database state and emitted events. A test that breaks when a private method is renamed, without any change in behavior, is a maintenance cost, not protection.
- **External services** are replaced at the boundary (HTTP fakes such as `Http::fake()` in Laravel or MSW in Node), including the failure cases: timeout, `500`, `429`.

### G1-4. Tests gate the merge, and stay deterministic

- **Rule**: The test suite runs in CI on every pull request and **blocks the merge** when it fails. A failing test is never skipped to ship faster.
- **Determinism**:
  - fixed clock in tests (`Carbon::setTestNow()`, `vi.useFakeTimers()`), never "wait 2 seconds";
  - each test creates its own data (factories) and runs in isolation (transaction rollback or fresh database);
  - no dependence on test order, on the real network or on the current date.
- **Flaky tests** are fixed or quarantined immediately, with an owner. A suite that "sometimes fails" teaches the team to ignore failures.
- **The CI gate** also runs: lint and type-check, `composer audit` / `npm audit`, secret scanning (G5), and the migration up/down check (G7). Performance budgets from prod-ready rule 25 join the same gate.

---

## G4 — Resilience & Graceful Degradation

Every dependency you do not control — a payment gateway, an email provider, a language model API, even your own Redis — will be slow or down at some point. The goal is that one failing dependency degrades one feature, never the whole system.

### G4-5. Timeouts on every network call

- **Rule**: Every call over the network has an explicit timeout: HTTP clients (connect and total), database statements, Redis, queues. Library defaults are often "wait forever".
- **Pattern**:

```php
// Laravel HTTP client
$response = Http::connectTimeout(2)->timeout(5)->get($url);
```

```ts
// Node 18+: AbortSignal.timeout
const res = await fetch(url, { signal: AbortSignal.timeout(5000) });
```

```sql
-- PostgreSQL: cap any single statement of this session or role
SET statement_timeout = '5s';
```

- **Budget**: the timeouts of inner calls must be shorter than the timeout of the request that waits for them; otherwise the outer request times out first and the inner work continues for nothing.
- **Streaming** responses (for example, from a language model) are measured as time to first byte and time between chunks, not total duration.

### G4-6. Retries only on idempotent operations, with backoff and jitter

- **Rule**: Retry only operations that are safe to repeat: reads, and writes protected by an idempotency key (G2). Never blindly retry a payment or an email send.
- **What to retry**: network errors, timeouts, `408`, `429`, `502`, `503`, `504`. **Never** other `4xx` (the request is wrong; repeating it will not help).
- **How**: a small maximum number of attempts (2–3), **exponential backoff with jitter**, and respect for the `Retry-After` header. Without jitter, every client retries at the same moment and overloads the recovering service again.

```ts
// TypeScript: exponential backoff with full jitter
async function withRetry<T>(fn: () => Promise<T>, attempts = 3, baseMs = 200): Promise<T> {
  for (let attempt = 1; ; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt >= attempts || !isRetryable(err)) throw err;
      const cap = baseMs * 2 ** (attempt - 1);
      await new Promise((r) => setTimeout(r, Math.random() * cap));
    }
  }
}
```

```php
// Laravel: retry with growing delay, only for retryable failures
$response = Http::timeout(5)
    ->retry(3, fn (int $attempt) => random_int(0, 200 * (2 ** ($attempt - 1))),
        fn (Throwable $e) => $e instanceof ConnectionException
            || ($e instanceof RequestException && in_array($e->response->status(), [429, 502, 503, 504], true)))
    ->get($url);
```

- **Do not stack retries**: if the HTTP client, the service and the queue all retry 3 times, one failure becomes 27 calls. Retry at one layer.

### G4-7. Circuit breaker: closed, open, half-open

- **Rule**: Calls to an external dependency go through a circuit breaker. After repeated failures it **opens** and fails immediately for a cool-down period, instead of making every request wait for the timeout. Then it lets a trial call through (**half-open**): success closes it, failure opens it again.
- **Pattern** (one breaker per dependency):

```ts
type State = 'closed' | 'open' | 'half-open';

export class CircuitBreaker {
  private state: State = 'closed';
  private failures = 0;
  private openedAt = 0;

  constructor(private readonly threshold = 5, private readonly coolDownMs = 30_000) {}

  async call<T>(fn: () => Promise<T>, fallback: () => T | Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.openedAt < this.coolDownMs) return fallback();
      this.state = 'half-open'; // let one trial call through
    }
    try {
      const result = await fn();
      this.state = 'closed';
      this.failures = 0;
      return result;
    } catch {
      this.failures++;
      if (this.state === 'half-open' || this.failures >= this.threshold) {
        this.state = 'open';
        this.openedAt = Date.now();
      }
      return fallback();
    }
  }
}

// Usage
const shippingBreaker = new CircuitBreaker();
const quote = await shippingBreaker.call(
  () => getShippingQuote(order, { signal: AbortSignal.timeout(3000) }),
  () => ({ status: 'unavailable' as const }),
);
```

- **Notes**:
  - This in-memory breaker counts failures **per instance**. That is usually fine; when many instances must share the state, keep the counters in Redis or use a library (for example, `opossum` in Node).
  - In PHP, each request starts with no memory, so the breaker state must live in a shared cache (Redis, `Cache::increment` with a TTL).
  - Record every state change in the logs and metrics (G6): an open circuit is an incident signal.

### G4-8. An explicit fallback for every dependency

- **Rule**: For each external dependency, decide in advance what the user sees when it fails. "Throw a 500" is not a fallback.
- **Typical fallbacks**:

| Dependency fails | Fallback |
|---|---|
| Recommendations, analytics, non-essential widgets | Hide the section; the page still works |
| Price or exchange rate service | Last known value from cache, labeled with its time |
| Shipping quote | "Shipping calculated at checkout" and continue |
| Email or SMS provider | Queue the message for later delivery (idempotent, G2) |
| Payment gateway | Clear message "payment temporarily unavailable", nothing charged, order kept as pending |
| Redis cache | Read from the database with a short timeout (cache is an optimization, not a dependency) |

- **Be honest with the user**: a degraded response says it is degraded. Never show stale or partial data as if it were complete and current, and never pretend a payment succeeded.
- **Mark degraded responses** in logs and metrics, so degradation is visible to the team and not silent.

### G4-9. Bulkheads and dead-letter queues

- **Bulkheads**: isolate resources so one slow dependency cannot exhaust everything. Separate queues and workers per kind of work (emails do not block payments), concurrency limits per external service, and separate connection pools where needed.
- **Dead-letter queues**: after the maximum number of attempts, a failed job goes to a dead-letter queue (or `failed_jobs` in Laravel) with its error and trace ID, instead of retrying forever or disappearing. Someone is alerted, and the job can be inspected and replayed — safely, because jobs are idempotent (G2).

```php
// Laravel job
class SendInvoiceEmail implements ShouldQueue
{
    public int $tries = 5;
    public array $backoff = [10, 30, 120, 600]; // seconds between attempts

    public function failed(Throwable $e): void
    {
        Log::error('invoice_email_failed', ['invoice_id' => $this->invoiceId, 'error' => $e->getMessage()]);
    }
}
```

---

## G6 — Observability

When something breaks in production, the question is "what happened to this request?". Without structured logs and a trace ID, the answer is a guess.

### G6-10. Structured JSON logs

- **Rule**: Logs are JSON objects, one per line, with a stable set of fields — not free text. Every entry has at least: `timestamp` (UTC), `level`, `message`, `trace_id`, and the relevant context (`tenant_id`, `user_id`, `route`, `status`, `duration_ms`).
- **Anti-pattern**: `console.log('error!!', err)` or `Log::info("User $email paid $amount")`.
- **Pattern**:

```ts
// Node (pino)
import pino from 'pino';
export const logger = pino({ level: process.env.LOG_LEVEL ?? 'info' });

logger.info({ traceId, tenantId, orderId, durationMs }, 'order_created');
```

```php
// Laravel: config/logging.php — JSON output
'json' => [
    'driver' => 'monolog',
    'handler' => Monolog\Handler\StreamHandler::class,
    'with' => ['stream' => 'php://stderr'],
    'formatter' => Monolog\Formatter\JsonFormatter::class,
],

// In code: a stable message plus structured context
Log::info('order_created', ['order_id' => $order->id, 'total_cents' => $order->total_cents]);
```

- **Levels mean something**: `error` = needs action (someone may be paged); `warn` = degraded but handled (fallback used, retry happened); `info` = business events (order created, user signed in); `debug` = off in production.
- **Stable messages**: use event names (`order_created`, `payment_failed`) so logs can be searched and counted; put variable data in fields.

### G6-11. A trace ID on every request, propagated everywhere

- **Rule**: Every incoming request gets a trace ID: accept the incoming W3C `traceparent` or `X-Request-Id` header if present, otherwise generate one. Add it to every log line of that request, return it in a response header, pass it to outgoing HTTP calls, and put it in every job dispatched by the request.
- **Pattern (Node)**:

```ts
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

const requestContext = new AsyncLocalStorage<{ traceId: string }>();

export function withTrace(req: Request, handler: () => Promise<Response>) {
  const traceId = req.headers.get('x-request-id') ?? randomUUID();
  return requestContext.run({ traceId }, async () => {
    const res = await handler();
    res.headers.set('x-request-id', traceId);
    return res;
  });
}

// Anywhere in the request: logs and outgoing calls carry the same id
export const log = () => logger.child({ traceId: requestContext.getStore()?.traceId });
```

- **Pattern (Laravel)**:

```php
// app/Http/Middleware/AssignTraceId.php
public function handle(Request $request, Closure $next)
{
    $traceId = $request->header('X-Request-Id') ?: (string) Str::uuid();

    Context::add('trace_id', $traceId); // Laravel 11+: added to every log entry and carried into queued jobs

    $response = $next($request);
    $response->headers->set('X-Request-Id', $traceId);
    return $response;
}
// Before Laravel 11: use Log::withContext(['trace_id' => $traceId]) and pass the id in each job's constructor.
```

- **Validate incoming ids**: accept only a reasonable format and length, so a client cannot inject arbitrary content into your logs.
- **Show the id on error pages** ("Error reference: 7f3c…"), so a user report leads straight to the logs.
- **Distributed tracing**: for systems with several services, OpenTelemetry carries the same trace across services and records the duration of each step.

### G6-12. No secrets or personal data in logs

- **Rule**: Never log passwords, tokens, API keys, full card numbers, session cookies, or full request bodies. Log identifiers (`user_id`), not personal data (email, document numbers, addresses), unless there is a clear need and a retention policy.
- **Pattern**: a redaction list in the logger configuration as a safety net:

```ts
const logger = pino({
  redact: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token', '*.cardNumber'],
});
```

- **Retention**: logs have a defined retention period and access control. Logs are data too, subject to privacy rules (such as GDPR or LGPD).

### G6-13. Error tracking, metrics and alerts

- **Error tracking**: unhandled exceptions are sent to an error tracker (Sentry or similar) with the trace ID, route, release version and user id. Production error responses show a generic message plus the trace ID, never the stack trace (G5).
- **Metrics per route** — the RED method: **R**ate (requests per second), **E**rrors (error rate), **D**uration (p50/p95/p99 latency). Plus queue depth, failed jobs, circuit breaker state and database pool usage.
- **Alerts on symptoms users feel** (error rate above a threshold, p95 latency up, queue not draining, a circuit open for minutes), not on every single error line. Every alert has an owner and a first action.
- **Health endpoints**: readiness and liveness checks (G7) are monitored too.
- **Frontend**: real-user metrics from prod-ready rule 25 report to the same observability backend, tagged with the same trace or session id, so a slow page can be followed into the backend request.

---

## Reliability checklist

- [ ] The bug fix includes a test that failed before the fix.
- [ ] Auth, permissions, money, idempotency, constraints and migrations have automated tests.
- [ ] Database tests run on the real engine; external services are faked at the boundary, including failures.
- [ ] The suite runs in CI, blocks the merge, and is deterministic.
- [ ] Every network call has a timeout shorter than its caller's.
- [ ] Retries only on idempotent operations and retryable errors, with backoff and jitter, at one layer only.
- [ ] External dependencies sit behind a circuit breaker with an explicit, honest fallback.
- [ ] Work is isolated in separate queues; failed jobs go to a dead-letter queue with an alert.
- [ ] Logs are JSON, leveled, with stable event names and the trace ID.
- [ ] The trace ID is accepted or generated, returned to the client, and propagated to calls and jobs.
- [ ] No secrets or unnecessary personal data in logs.
- [ ] Errors are tracked, RED metrics exist per route, and alerts target user-facing symptoms.
