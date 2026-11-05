# Security Rules — G5: Authorization, IDOR Prevention & Input Sanitization

**Level:** Always. **Precedence:** highest — no performance, cache or UX rule overrides it.

Most production data leaks are not exotic exploits. They are an endpoint that forgot to check *who owns this record*, a query that filters by tenant after loading everything, or a string concatenated into SQL. This file turns those into hard rules.

---

## Index

1. [Authenticate every request, deny by default](#1-authenticate-every-request-deny-by-default)
2. [Authorize the action on the specific record (RBAC + ownership)](#2-authorize-the-action-on-the-specific-record-rbac--ownership)
3. [Prevent IDOR: scope every query by tenant and owner](#3-prevent-idor-scope-every-query-by-tenant-and-owner)
4. [Validate every input against an explicit schema](#4-validate-every-input-against-an-explicit-schema)
5. [Parameterize all queries](#5-parameterize-all-queries)
6. [Encode output and sanitize rich content](#6-encode-output-and-sanitize-rich-content)
7. [Prevent mass assignment and over-exposure](#7-prevent-mass-assignment-and-over-exposure)
8. [Keep secrets out of code, logs and responses](#8-keep-secrets-out-of-code-logs-and-responses)
9. [Rate limit and protect sessions](#9-rate-limit-and-protect-sessions)

---

### 1. Authenticate every request, deny by default

- **Rule**: Every route requires authentication unless it is explicitly marked public. The default is deny.
- **Anti-pattern**: Adding `auth` middleware route by route and forgetting one.
- **Pattern**: Apply authentication to the whole route group and opt *out* the few public routes (login, health check, public pages).

```php
// Laravel: routes/api.php — everything is protected unless listed outside the group
Route::middleware(['auth:sanctum'])->group(function () {
    Route::apiResource('orders', OrderController::class);
});
Route::post('/login', [AuthController::class, 'login']); // explicitly public
```

- **Verify**: A test lists all routes and fails if a non-whitelisted route has no auth middleware.

---

### 2. Authorize the action on the specific record (RBAC + ownership)

- **Rule**: Checking the role is not enough. The server checks that *this user* may perform *this action* on *this record*. Roles answer "can editors edit orders?"; ownership answers "can this editor edit order #42?". Both are required.
- **Anti-pattern**: `if (user.role === 'admin' || user.role === 'editor')` and then loading the order by the id from the URL.
- **Pattern**: A central policy per resource, called in every handler. Never authorize only in the frontend (hidden buttons are not security).

```php
// Laravel: app/Policies/OrderPolicy.php
class OrderPolicy
{
    public function update(User $user, Order $order): bool
    {
        return $user->tenant_id === $order->tenant_id
            && ($user->hasRole('admin') || $order->owner_id === $user->id);
    }
}

// Controller
public function update(UpdateOrderRequest $request, Order $order)
{
    $this->authorize('update', $order);
    $order->update($request->validated());
    return new OrderResource($order);
}
```

```ts
// TypeScript: one place that decides, used by every handler
type Action = 'read' | 'update' | 'delete';

export function can(user: SessionUser, action: Action, order: { tenantId: string; ownerId: string }) {
  if (user.tenantId !== order.tenantId) return false;
  if (user.role === 'admin') return true;
  if (action === 'read') return true;
  return order.ownerId === user.id;
}
```

- **Verify**: For each protected action, a test where a user of another tenant and a user without permission both receive `403` or `404`.

---

### 3. Prevent IDOR: scope every query by tenant and owner

- **Rule**: The tenant (and, where it applies, the owner) is part of the `WHERE` clause of every query. Never load a record by id alone and check access afterwards; never filter the result in application code.
- **Anti-pattern (IDOR)**: `GET /api/invoices/1043` → `SELECT * FROM invoices WHERE id = 1043`. Changing the number in the URL shows another customer's invoice.
- **Pattern**:

```sql
-- SQL: tenant in the WHERE, always
SELECT id, number, total_cents, status
FROM invoices
WHERE id = $1 AND tenant_id = $2;
```

```php
// Laravel: a global scope makes forgetting impossible
class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where($model->getTable() . '.tenant_id', auth()->user()->tenant_id);
    }
}
// In each tenant-owned model:
protected static function booted(): void { static::addGlobalScope(new TenantScope); }
```

```ts
// Prisma: the tenant comes from the session, never from the request body or URL
const invoice = await prisma.invoice.findFirst({
  where: { id: params.id, tenantId: session.tenantId },
  select: { id: true, number: true, totalCents: true, status: true },
});
if (!invoice) return new Response(null, { status: 404 }); // 404, not 403: do not confirm it exists
```

- **Also**:
  - Respond `404` for records outside the caller's scope, so ids cannot be probed.
  - Prefer non-sequential public ids (UUID v7, ULID) for externally visible resources. This reduces enumeration but **does not replace** the scope check.
  - Cache keys, background job payloads, file paths and signed URLs carry the same scope. A job that receives `invoiceId` re-checks the tenant before acting.
  - For PostgreSQL, Row-Level Security is a strong second layer for multi-tenant tables.
- **Verify**: An automated test requests another tenant's record id and expects `404`.

---

### 4. Validate every input against an explicit schema

- **Rule**: Every input — body, query string, route params, headers, uploaded files, webhook payloads, and messages from queues — is validated against an explicit schema with type, length, format and allowed values. Reject unknown fields.
- **Anti-pattern**: Reading `req.body.status` and writing it to the database.
- **Pattern**:

```ts
// TypeScript (zod)
import { z } from 'zod';

const UpdateOrder = z.object({
  status: z.enum(['pending', 'paid', 'shipped', 'cancelled']),
  note: z.string().trim().max(500).optional(),
}).strict(); // unknown fields are rejected

const parsed = UpdateOrder.safeParse(await req.json());
if (!parsed.success) return Response.json({ error: 'invalid_input' }, { status: 422 });
```

```php
// Laravel: app/Http/Requests/UpdateOrderRequest.php
public function rules(): array
{
    return [
        'status' => ['required', Rule::in(['pending', 'paid', 'shipped', 'cancelled'])],
        'note'   => ['nullable', 'string', 'max:500'],
    ];
}
```

- **Files**: check the real content type (magic bytes, not the extension), cap the size, rename on storage, store outside the web root or in object storage, and strip metadata from images.
- **Verify**: Tests send a wrong type, an oversized string and an unknown field, and expect `422`.

---

### 5. Parameterize all queries

- **Rule**: User input never becomes part of a SQL string. Use bound parameters or the ORM's query builder. Dynamic column names or sort directions come from an allow-list.
- **Anti-pattern**:

```php
DB::select("SELECT id, name FROM users WHERE name LIKE '%" . $request->q . "%'"); // SQL injection
```

- **Pattern**:

```php
// Laravel
User::where('name', 'like', '%' . $request->validated('q') . '%')->get(['id', 'name']);

// Sort column from an allow-list, never raw
$sort = in_array($request->sort, ['name', 'created_at'], true) ? $request->sort : 'created_at';
```

```ts
// Prisma raw SQL: tagged template = parameters, never $queryRawUnsafe with user input
const rows = await prisma.$queryRaw`SELECT id, name FROM "User" WHERE "tenantId" = ${tenantId}`;
```

- **Verify**: Code review rule plus a lint/grep check that rejects string concatenation into raw SQL and `$queryRawUnsafe`/`DB::raw` with request data.

---

### 6. Encode output and sanitize rich content

- **Rule**: Data is encoded for the context where it is rendered (HTML, attribute, URL, JavaScript). Framework templates do this by default — never bypass it with user data. User-provided HTML or Markdown, and text generated by language models, is sanitized with an allow-list before rendering.
- **Anti-pattern**: `dangerouslySetInnerHTML={{ __html: comment.body }}` or `{!! $comment->body !!}`.
- **Pattern**: Render as text (`{comment.body}`, `{{ $comment->body }}`). When rich content is required, sanitize with an allow-list library (DOMPurify in the browser, HTML Purifier in PHP) and set a Content-Security-Policy.
- **Headers**: `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Strict-Transport-Security`. Cookies: `HttpOnly`, `Secure`, `SameSite=Lax` or stricter. Forms with cookie sessions need CSRF protection.

---

### 7. Prevent mass assignment and over-exposure

- **Rule (input)**: Only explicitly allowed fields can be written. Fields like `role`, `tenant_id`, `owner_id`, `is_admin`, `price` are never taken from the request.
- **Rule (output)**: Responses go through an explicit output shape (DTO, API Resource, `select`). Never return the whole model.
- **Pattern**:

```php
// Laravel: write only validated fields; set ownership from the session
$order = Order::create([
    ...$request->validated(),
    'tenant_id' => $request->user()->tenant_id,
    'owner_id'  => $request->user()->id,
]);
return new OrderResource($order); // Resource lists exactly what leaves the server
```

```ts
// Prisma: explicit select is also a security boundary
select: { id: true, name: true, avatarUrl: true } // never passwordHash, tokens, internal flags
```

- **Verify**: A test sends `{"role":"admin"}` in an update and asserts the role did not change; a snapshot test of the response shape catches new fields leaking out.

---

### 8. Keep secrets out of code, logs and responses

- **Rule**: Secrets (API keys, database passwords, signing keys) come from environment variables or a secret manager, never from the repository. They are never logged and never sent to the client.
- **Pattern**:
  - `.env` is in `.gitignore`; a `.env.example` documents the variable names with empty values.
  - Secret scanning runs in CI (for example, gitleaks).
  - Error responses in production never include stack traces, SQL or internal paths: return a generic message plus the trace ID (G6).
  - Passwords are hashed with a slow algorithm (bcrypt, argon2id) — never encrypted, never stored in plain text.
  - Dependencies are audited (`npm audit`, `composer audit`) and lockfiles are committed.

---

### 9. Rate limit and protect sessions

- **Rule**: Login, password reset, signup, search and any expensive endpoint are rate limited per user and per IP.
- **Pattern**:

```php
// Laravel
RateLimiter::for('login', fn (Request $r) => Limit::perMinute(5)->by($r->input('email') . '|' . $r->ip()));
Route::post('/login', ...)->middleware('throttle:login');
```

- **Sessions and tokens**:
  - Regenerate the session id on login.
  - Short-lived access tokens (JWTs are **signed**, not encrypted — never put secrets in them).
  - Rotating refresh tokens, revocable server-side.
  - Log out invalidates the session on the server.
- **Responses**: authenticated pages and APIs use `Cache-Control: private, no-store`.

---

## Security checklist

- [ ] Route group protected by default; public routes listed explicitly.
- [ ] Policy check on the specific record in every handler.
- [ ] Tenant (and owner) in the `WHERE` of every query; out-of-scope ids return `404`.
- [ ] Input validated by schema; unknown fields rejected; files checked by content.
- [ ] No string-built SQL; dynamic sort/columns from an allow-list.
- [ ] Output encoded; rich content sanitized; security headers set.
- [ ] Sensitive fields never accepted from input nor returned in output.
- [ ] No secrets in code, logs or error responses.
- [ ] Rate limits on auth and expensive endpoints; sessions regenerated and revocable.
