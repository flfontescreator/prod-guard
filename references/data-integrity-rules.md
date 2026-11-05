# Data Integrity Rules — G2 Idempotency, G3 Constraints & Indexes, G7 Migrations & Zero-Downtime

**Precedence:** second only to security. A slow page can be fixed later; a corrupted or duplicated record often cannot.

---

## Index

**G2 — Idempotent critical operations** *(On writes)*
1. [Idempotency keys for client-retried writes](#g2-1-idempotency-keys-for-client-retried-writes)
2. [Natural unique keys and upserts](#g2-2-natural-unique-keys-and-upserts)
3. [Idempotent jobs and webhooks](#g2-3-idempotent-jobs-and-webhooks)
4. [Enqueue after commit (transactional outbox)](#g2-4-enqueue-after-commit-transactional-outbox)

**G3 — Database integrity** *(Always)*
5. [Constraints enforce the invariants](#g3-5-constraints-enforce-the-invariants)
6. [Correct types for money, time and identifiers](#g3-6-correct-types-for-money-time-and-identifiers)
7. [Indexes for every foreign key, filter and sort](#g3-7-indexes-for-every-foreign-key-filter-and-sort)
8. [Transactions and concurrency control](#g3-8-transactions-and-concurrency-control)

**G7 — Reversible migrations & zero-downtime** *(On schema changes)*
9. [Every migration has a working down](#g7-9-every-migration-has-a-working-down)
10. [Expand → migrate → contract](#g7-10-expand--migrate--contract)
11. [No long locks: online DDL and batched backfills](#g7-11-no-long-locks-online-ddl-and-batched-backfills)
12. [Zero-downtime deploy mechanics](#g7-12-zero-downtime-deploy-mechanics)

---

## G2 — Idempotent Critical Operations

Networks fail after the server has done the work but before the client receives the answer. The client, the queue or the webhook sender then retries. Without idempotency, that retry charges the card twice, creates two orders or sends two emails.

> **Deduplication is not idempotency.** Deduplicating identical requests on the client (prod-ready rule 4) only saves bandwidth on reads. Protection against repeated writes must live on the server.

### G2-1. Idempotency keys for client-retried writes

- **Rule**: Writes with real-world effects (payments, orders, transfers, bookings, sending messages) accept an `Idempotency-Key` header. The server stores the key with a unique constraint and returns the original result for a repeated key.
- **Behavior**:
  - First request with the key → process and store the response.
  - Same key, same payload, finished → return the stored response; do nothing again.
  - Same key, still processing → `409 Conflict`.
  - Same key, different payload → `422`; the key was reused by mistake.
  - Keys expire after a fixed window (for example, 24 hours).
- **Table**:

```sql
CREATE TABLE idempotency_keys (
  tenant_id      BIGINT       NOT NULL,
  key            VARCHAR(255) NOT NULL,
  request_hash   CHAR(64)     NOT NULL,          -- SHA-256 of method + path + body
  status         VARCHAR(16)  NOT NULL CHECK (status IN ('processing', 'completed')),
  response_code  SMALLINT,
  response_body  TEXT,
  created_at     TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (tenant_id, key)
);
```

```ts
// TypeScript / Prisma
export async function withIdempotency(
  tenantId: string, key: string, requestHash: string,
  handler: () => Promise<{ status: number; body: unknown }>,
) {
  try {
    await prisma.idempotencyKey.create({
      data: { tenantId, key, requestHash, status: 'processing' },
    });
  } catch (e: any) {
    if (e.code !== 'P2002') throw e; // P2002 = unique constraint violation
    const existing = await prisma.idempotencyKey.findUnique({
      where: { tenantId_key: { tenantId, key } },
    });
    if (!existing) throw e;
    if (existing.requestHash !== requestHash) return { status: 422, body: { error: 'idempotency_key_reused' } };
    if (existing.status === 'processing') return { status: 409, body: { error: 'request_in_progress' } };
    return { status: existing.responseCode!, body: JSON.parse(existing.responseBody!) };
  }

  try {
    const result = await handler();
    await prisma.idempotencyKey.update({
      where: { tenantId_key: { tenantId, key } },
      data: { status: 'completed', responseCode: result.status, responseBody: JSON.stringify(result.body) },
    });
    return result;
  } catch (err) {
    // Failed before completing: release the key so the client can retry
    await prisma.idempotencyKey.delete({ where: { tenantId_key: { tenantId, key } } });
    throw err;
  }
}
```

- **Client side**: the client generates the key once per user action (`crypto.randomUUID()`) and reuses the **same** key on every retry of that action.
- **Downstream**: pass the key on to providers that support it (most payment gateways accept an idempotency key), so the external charge is also protected.

### G2-2. Natural unique keys and upserts

- **Rule**: When the data has a natural identity (an external order number, one subscription per user per plan, one vote per user per poll), the database enforces it with a `UNIQUE` constraint and the write uses an upsert or handles the conflict.
- **Pattern**:

```sql
-- PostgreSQL
INSERT INTO subscriptions (user_id, plan_id, status)
VALUES ($1, $2, 'active')
ON CONFLICT (user_id, plan_id) DO UPDATE SET status = EXCLUDED.status;

-- MySQL
INSERT INTO subscriptions (user_id, plan_id, status)
VALUES (?, ?, 'active')
ON DUPLICATE KEY UPDATE status = VALUES(status);
```

```php
// Laravel
Subscription::upsert(
    [['user_id' => $userId, 'plan_id' => $planId, 'status' => 'active']],
    uniqueBy: ['user_id', 'plan_id'],
    update: ['status'],
);
```

- **Anti-pattern**: "check then insert" in application code (`if (!exists) insert`) — two concurrent requests both see "not exists" and both insert.
- **Relative updates** (`balance = balance + 10`) are not idempotent by nature. Record the movement as a row with a unique id (a ledger entry) and derive the balance from it, or guard the update with the idempotency key.

### G2-3. Idempotent jobs and webhooks

- **Rule**: Queues and webhook senders deliver **at least once**. Every job and webhook handler can run twice with the same input without a second effect.
- **Pattern**: record each processed message id with a unique constraint, in the same transaction as the effect.

```php
// Laravel: webhook handler
public function handle(Request $request)
{
    $eventId = $request->input('id');

    DB::transaction(function () use ($eventId, $request) {
        $inserted = DB::table('processed_events')->insertOrIgnore([
            'event_id' => $eventId,
            'processed_at' => now(),
        ]);
        if ($inserted === 0) {
            return; // already processed
        }
        $this->applyPayment($request->input('data'));
    });

    return response()->noContent(); // 2xx so the sender stops retrying
}
```

- **Also**: verify the webhook signature before anything else (G5); make the job's effect conditional on current state (`UPDATE orders SET status = 'paid' WHERE id = ? AND status = 'pending'`); configure a maximum number of attempts and a dead-letter queue (G4).

### G2-4. Enqueue after commit (transactional outbox)

- **Rule**: A job or event is published only after the transaction that created its data has committed. Otherwise a worker may pick up a job for a row that was rolled back, or that is not visible yet.
- **Pattern**:

```php
// Laravel: dispatch only after the surrounding transaction commits
DB::transaction(function () use ($data) {
    $order = Order::create($data);
    SendOrderConfirmation::dispatch($order->id)->afterCommit();
});
```

- **Transactional outbox** (when the event must never be lost): write the event to an `outbox` table in the same transaction as the data; a separate dispatcher reads unpublished rows, publishes them and marks them as sent. Consumers are idempotent (G2-3), because the dispatcher may publish a row twice.

---

## G3 — Database Integrity

The application validates for good error messages; the database enforces for correctness. Application checks are bypassed by concurrency, scripts, other services and bugs. Constraints are not.

### G3-5. Constraints enforce the invariants

- **Rule**: Every invariant that can be expressed in the schema is expressed there:
  - `NOT NULL` on every column that must have a value;
  - `UNIQUE` on natural identities (email per tenant, document number, external ids);
  - `FOREIGN KEY` on every reference, with an explicit `ON DELETE` (`RESTRICT` by default; `CASCADE` only for true child rows; `SET NULL` when the reference is optional);
  - `CHECK` for ranges and allowed values (`quantity > 0`, `status IN (...)`).
- **Pattern**:

```sql
CREATE TABLE order_items (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id    BIGINT  NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id  BIGINT  NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
  quantity    INTEGER NOT NULL CHECK (quantity > 0),
  unit_cents  BIGINT  NOT NULL CHECK (unit_cents >= 0),
  UNIQUE (order_id, product_id)
);
```

```php
// Laravel migration
Schema::create('order_items', function (Blueprint $table) {
    $table->id();
    $table->foreignId('order_id')->constrained()->cascadeOnDelete();
    $table->foreignId('product_id')->constrained()->restrictOnDelete();
    $table->unsignedInteger('quantity');
    $table->unsignedBigInteger('unit_cents');
    $table->unique(['order_id', 'product_id']);
});
// CHECK constraints: Blueprint has no portable helper; add them with DB::statement() after the table.
```

```prisma
// Prisma schema
model OrderItem {
  id        BigInt  @id @default(autoincrement())
  orderId   BigInt
  productId BigInt
  quantity  Int
  unitCents BigInt
  order     Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product   Product @relation(fields: [productId], references: [id], onDelete: Restrict)

  @@unique([orderId, productId])
  @@index([productId])
}
// CHECK constraints are not expressible in the Prisma schema: add them in the generated migration SQL.
```

- **Unique with soft deletes**: a plain `UNIQUE (tenant_id, email)` blocks re-creating a soft-deleted user. In PostgreSQL use a partial unique index: `CREATE UNIQUE INDEX ... ON users (tenant_id, email) WHERE deleted_at IS NULL;`. MySQL has no partial indexes: use a generated column that is `NULL` for deleted rows and put it in the unique key.
- **Multi-tenant**: unique constraints that represent business identity include `tenant_id` (`UNIQUE (tenant_id, email)`), otherwise one customer blocks another.

### G3-6. Correct types for money, time and identifiers

- **Money**: integer minor units (`BIGINT` cents) or `DECIMAL(19,4)`. Never `FLOAT`/`DOUBLE`: `0.1 + 0.2` is not `0.3` in binary floating point. Store the currency next to the amount when more than one is possible.
- **Time**: store instants in UTC (`TIMESTAMPTZ` in PostgreSQL; `DATETIME` written in UTC in MySQL). Convert to the user's time zone only for display. Dates without time (birthdays, due dates) use `DATE`.
- **Identifiers**: `BIGINT` for internal keys (an `INT` overflows at about 2.1 billion). For public ids, UUID v7 or ULID keep index locality better than random UUID v4.
- **Status fields**: a `CHECK` or an enum type with the allowed values, not a free text column.

### G3-7. Indexes for every foreign key, filter and sort

- **Rule**: Every foreign key column, every column used in a frequent `WHERE`, and every `ORDER BY` used for pagination has a matching index. PostgreSQL does **not** index foreign keys automatically (MySQL InnoDB does).
- **Composite index order**: equality columns first, then the range or sort columns, in the same order and direction as the query.

```sql
-- Query: WHERE tenant_id = ? ORDER BY created_at DESC, id DESC LIMIT 20
CREATE INDEX orders_tenant_created_id ON orders (tenant_id, created_at DESC, id DESC);
```

- **Cursor pagination**: the sort key must be unique and stable — `(created_at, id)`, never `created_at` alone, or rows with the same timestamp are repeated or skipped between pages.
- **Text search**: `LIKE '%term%'` cannot use a B-tree index. Use a trigram GIN index (`pg_trgm`) in PostgreSQL or a `FULLTEXT` index in MySQL, or a search engine for large datasets.
- **Do not over-index**: every index slows writes and uses memory. Add indexes for real query patterns, and remove unused ones.
- **Verify**: run `EXPLAIN` (PostgreSQL: `EXPLAIN ANALYZE`) on the main queries and confirm an index scan instead of a sequential/full table scan on large tables.

### G3-8. Transactions and concurrency control

- **Rule**: A business operation that writes more than one row runs in a single transaction: all or nothing.
- **Concurrent updates of the same row** (stock, seats, balances) need explicit control:

```sql
-- Pessimistic: lock the row for the rest of the transaction
BEGIN;
SELECT quantity FROM stock WHERE product_id = $1 FOR UPDATE;
UPDATE stock SET quantity = quantity - $2 WHERE product_id = $1;
COMMIT;

-- Or a single conditional statement, which needs no explicit lock
UPDATE stock SET quantity = quantity - $2
WHERE product_id = $1 AND quantity >= $2;  -- 0 rows affected = not enough stock
```

```php
// Laravel: optimistic locking with a version column
$updated = Order::where('id', $id)
    ->where('version', $request->integer('version'))
    ->update([...$changes, 'version' => DB::raw('version + 1')]);

if ($updated === 0) {
    abort(409, 'The record was changed by someone else. Reload and try again.');
}
```

- **Keep transactions short**: never call external APIs or send emails inside a database transaction. Commit first, then trigger the side effect (G2-4).

---

## G7 — Reversible Migrations & Zero-Downtime Deployment

During a deploy, old and new versions of the code run at the same time against the **same** database. Every schema change must work with both.

### G7-9. Every migration has a working down

- **Rule**: Every migration can be reverted. The `down` is written together with the `up` and tested (run `up`, `down`, `up` again in CI or locally).

```php
// Laravel
public function up(): void
{
    Schema::table('orders', function (Blueprint $table) {
        $table->string('tracking_code', 64)->nullable()->after('status');
    });
}

public function down(): void
{
    Schema::table('orders', function (Blueprint $table) {
        $table->dropColumn('tracking_code');
    });
}
```

- **Prisma Migrate** does not generate down migrations automatically. Generate the revert script when you create the migration (`prisma migrate diff ... --script`) and keep it next to the migration.
- **Destructive steps** (dropping a column or table) cannot restore data in `down`. Run them only in the contract phase (G7-10), after a backup, and mark them clearly as irreversible.
- **Never edit a migration that already ran in production**; create a new one.

### G7-10. Expand → migrate → contract

- **Rule**: A breaking change (rename, type change, split or merge of columns, new `NOT NULL`) is done in separate deploys, so no version of the code ever sees a schema it does not understand.
- **Example — renaming `orders.state` to `orders.status`**:
  1. **Expand** (deploy 1): add `status` as nullable. The code writes to **both** columns and still reads `state`.
  2. **Migrate**: backfill `status` from `state` in batches (G7-11).
  3. **Switch** (deploy 2): the code reads `status`, still writes both.
  4. **Contract** (deploy 3, after confirming nothing reads `state`): stop writing `state`, then drop it in a later migration.
- **Adding a `NOT NULL` column to a large table**: add it as nullable (or with a constant default), backfill, then enforce `NOT NULL`. In PostgreSQL, add `CHECK (col IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT`, then `SET NOT NULL`, to avoid a long full-table lock.
- **Removing a column**: first deploy code that no longer uses it (including ORM models that select all columns), then drop it.

### G7-11. No long locks: online DDL and batched backfills

- **Indexes on large tables**:
  - PostgreSQL: `CREATE INDEX CONCURRENTLY` (cannot run inside a transaction; in Laravel set `public $withinTransaction = false;` on the migration).
  - MySQL (InnoDB): `ALTER TABLE ... ADD INDEX ..., ALGORITHM=INPLACE, LOCK=NONE`.
- **Lock timeout**: in PostgreSQL, set `lock_timeout` (for example, `SET lock_timeout = '5s';`) before DDL, so a blocked migration fails fast instead of queueing every query behind it.
- **Backfills** run in batches by primary key, outside the deploy migration, with a pause between batches:

```php
// Laravel: batched backfill (artisan command or job, not a schema migration)
Order::whereNull('status')
    ->chunkById(1000, function ($orders) {
        Order::whereIn('id', $orders->pluck('id'))
            ->update(['status' => DB::raw('state')]);
        usleep(100_000); // let regular traffic breathe
    });
```

### G7-12. Zero-downtime deploy mechanics

- **Order**: run expand migrations **before** deploying the code that needs them; run contract migrations only **after** no running version needs the old schema.
- **Rolling deploys**: replace instances gradually behind the load balancer. This requires stateless instances (prod-ready rule 23).
- **Health checks**: a *readiness* check (can this instance take traffic? database reachable, migrations compatible) separate from a *liveness* check (is the process alive?). The load balancer routes only to ready instances.
- **Graceful shutdown**: on `SIGTERM`, stop accepting new requests, finish in-flight requests and jobs within a deadline, then exit. Queue workers finish the current job (`php artisan queue:work` handles this; Node needs a `process.on('SIGTERM', ...)` handler that closes the server and the pools).
- **Asset version skew**: keep the previous release's static assets available for some time, so users with the old HTML do not get `404` on old JavaScript chunks.
- **Feature flags** separate *deploy* from *release*: ship the code dark, enable it gradually, and turn it off without a rollback.
- **Rollback plan**: before each deploy, know whether the code can be rolled back alone. After the expand phase it can; after a contract migration, rolling back the code requires restoring the schema first.

---

## Data integrity checklist

- [ ] Writes with real-world effects accept an `Idempotency-Key`, stored under a unique constraint.
- [ ] Natural identities have `UNIQUE` constraints; no "check then insert".
- [ ] Jobs and webhooks record processed ids; effects are conditional on current state.
- [ ] Jobs are dispatched after commit; critical events use an outbox.
- [ ] `NOT NULL`, `UNIQUE`, `FOREIGN KEY` with explicit `ON DELETE`, and `CHECK` where they apply.
- [ ] Money in integer cents or `DECIMAL`; time in UTC; `BIGINT` keys.
- [ ] Indexes on foreign keys, filters and pagination order; cursor order has a unique tie-breaker.
- [ ] Multi-row writes in a transaction; concurrent updates guarded; no external calls inside transactions.
- [ ] Every migration has a tested `down`; breaking changes split into expand/migrate/contract.
- [ ] Large-table DDL is online; backfills are batched; deploys are rolling with graceful shutdown.
