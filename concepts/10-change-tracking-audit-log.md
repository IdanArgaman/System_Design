# Change Tracking / Audit Log in Node.js — Design, Pitfalls & Alternatives

> **Goal:** Record *who* changed *what*, *when*, with *old → new* values, without putting audit code in business services.
> **Main stack:** Node.js + Prisma + PostgreSQL + `AsyncLocalStorage`.
> **Also covered:** Postgres triggers, transactional outbox / domain events, CDC (Debezium), and Mongoose plugins.

---

## 0. TL;DR (the 60-second interview answer)

1. **Don't** write audit calls inside `updateProduct()`. It breaks SRP, gets duplicated in every service, and someone will forget to add it.
2. **Capture context once** at the edge (HTTP middleware / job runner) and store it in **`AsyncLocalStorage`** (`userId`, `requestId`, `ip`).
3. **Intercept writes in the data layer**, using one of these:
   - **Prisma Client Extension**: application-level, easy to demo, knows about the app's models.
   - **Postgres trigger + `set_config('app.user_id')`**: the strongest guarantee. It catches raw SQL, migrations, other services and admin consoles.
4. **Write the audit row in the same transaction** as the change (atomic), or use an **outbox** when audit events also feed other systems.
5. Know the edge cases: **bulk ops, nested writes, concurrent updates, `Date`/`Decimal`/`Json` diffing, PII redaction, append-only storage, retention.**

---

## 1. The problem

Enterprise systems need an audit trail for compliance (SOX, HIPAA, GDPR), debugging ("who set this price to 0?"), customer support, and undo/history features.

### The anti-pattern

```javascript
// ❌ Audit logic mixed into business logic
async updateProduct(id, data, userId) {           // userId leaks into every signature
  const before = await db.product.findUnique({ where: { id } });
  const after  = await db.product.update({ where: { id }, data });
  await db.auditLog.create({ data: { /* diff before/after */ } }); // copy-pasted into every service
  return after;
}
```

| Problem | Why it hurts |
|---|---|
| Violates **SRP** | The service now handles business rules, persistence and compliance |
| **Duplication** | The same code is repeated in `User`, `Order`, `Product` … and each copy drifts |
| **Opt-in** | A new endpoint that forgets the call has no audit, and nothing fails to warn you |
| **Signature pollution** | `userId` gets threaded through controller → service → repository |

### The goal

Business code stays clean. Auditing is **transparent**, **automatic (opt-out, not opt-in)**, **consistent**, and has access to **request context**.

---

## 2. Solution landscape

| # | Approach | Where it runs | Catches raw SQL / other services? | Knows the user? | Atomic with change? | Complexity |
|---|---|---|---|---|---|---|
| A | **ORM extension / hooks** (Prisma `$extends`, Mongoose plugin) | App | ❌ | ✅ via ALS | ✅ if wrapped in a tx | Low |
| B | **Repository / decorator** | App | ❌ | ✅ via ALS | ✅ | Low–Med (needs discipline) |
| C | **DB trigger + session variable** | DB | ✅ | ✅ via `set_config` | ✅ always | Med |
| D | **Transactional outbox / domain events** | App → broker | ❌ | ✅ | ✅ (event row in same tx) | Med–High |
| E | **CDC** (Debezium on the WAL → Kafka) | DB log → stream | ✅ | ⚠️ only if the row stores `modified_by` | ✅ (reads committed log) | High (infra) |
| F | **Event sourcing** | Architecture | n/a | ✅ | ✅ (events are the data) | Very high |

**Important distinction for interviews:**
- **Row-level diff audit** (A, B, C, E) answers *"what changed?"*, e.g. `price: 100 → 80`.
- **Domain-event audit** (D, F) answers *"why did it change?"*, e.g. `DiscountApplied{ campaignId }`.
- Mature systems often use **both**: triggers or an extension for compliance, and domain events for business meaning.

---

## 3. Context propagation with `AsyncLocalStorage`

The data layer needs to know **who** triggered the write, without receiving `userId` as a parameter.

`AsyncLocalStorage` (ALS) from `node:async_hooks` gives each async call chain its own store. This is Node's equivalent of thread-local storage. Everything started inside `als.run(store, fn)` (awaits, timers, promise chains) sees the same `store`. Concurrent requests stay isolated.

### `context.js`

```javascript
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

export const auditContext = new AsyncLocalStorage();

/** Read the current context (or a safe default for scripts / jobs). */
export const getAuditContext = () =>
  auditContext.getStore() ?? { userId: 'system', requestId: null, source: 'unknown' };

/** Run any unit of work (HTTP request, cron job, queue message) inside a context. */
export const runWithAuditContext = (ctx, fn) =>
  auditContext.run({ requestId: randomUUID(), source: 'unknown', ...ctx }, fn);
```

### Express middleware (after authentication)

```javascript
// ✅ userId comes from a VERIFIED identity (JWT/session), never from a client-controlled header.
app.use(authenticate); // sets req.user
app.use((req, res, next) => {
  runWithAuditContext(
    {
      userId: req.user?.id ?? 'anonymous',
      requestId: req.get('x-request-id') ?? undefined, // correlation id is fine to accept
      ip: req.ip,
      source: 'http',
    },
    next,
  );
});
```

### Background jobs / consumers need a context too

```javascript
queue.process('nightly-reprice', (job) =>
  runWithAuditContext({ userId: 'system:nightly-reprice', source: 'job', requestId: job.id }, () =>
    repriceAll(),
  ),
);
```

### Quick demo that ALS isolates concurrent requests (runnable as-is)

```javascript
import { AsyncLocalStorage } from 'node:async_hooks';
const als = new AsyncLocalStorage();

const deepInTheStack = async (tag) => {
  await new Promise((r) => setTimeout(r, Math.random() * 20)); // interleave the two "requests"
  console.log(tag, '->', als.getStore().userId);              // no parameters passed!
};

await Promise.all([
  als.run({ userId: 'alice' }, () => deepInTheStack('req1')),
  als.run({ userId: 'bob' },   () => deepInTheStack('req2')),
]);
// req1 -> alice
// req2 -> bob
```

### ALS gotchas worth mentioning
- Use **`als.run()`**, not `enterWith()`. `enterWith` leaks the store into the rest of the current synchronous execution.
- Context can be **lost** in libraries that use their own callback queues or connection pools (older callback-based drivers, some `EventEmitter` patterns). Fix: `AsyncResource.bind(fn)`, or promisified APIs.
- Performance overhead is small in modern Node, and recent Node versions reimplemented ALS on `AsyncContextFrame`. It is still worth measuring on a hot path.
- The original guide read `userId` from an `x-user-id` header. That is a **security bug**, because any client can impersonate any user in the audit trail.

---

## 4. Approach A (primary demo): Prisma Client Extension

### 4.1 Review of the original implementation: bugs & gaps

| # | Issue in the original | Consequence | Fix |
|---|---|---|---|
| 1 | `oldObj[key] !== newObj[key]` for all fields | **`Date`, `Decimal`, `Json`, `Bytes` fields always show as "changed"**, because they are different object instances with equal values. Verified: an unchanged `createdAt` shows up in the diff | Normalize values, then compare with `util.isDeepStrictEqual` |
| 2 | `prisma[model.toLowerCase()]` | `OrderItem` → `orderitem` → **undefined**. Prisma delegates are camelCase (`orderItem`) | lower-case the first character only |
| 3 | Pre-read only when `args.where.id` exists | Updates by a unique key (`where: { sku }`) or a compound key are **silently not audited** | `update` requires a unique `where`, so pass `args.where` straight to `findUnique` |
| 4 | Read-before-write outside a transaction | **Race:** another request can update between the read and the write, so the logged `old` values are wrong | Run read + write + audit in one tx at `RepeatableRead`, and retry on conflict |
| 5 | Audit write outside the transaction, errors swallowed by `.catch` | Data changes with **no audit record**. Fine for "nice-to-have" logs, but a compliance failure | Same transaction: either both commit or neither does |
| 6 | Only `update` is intercepted | `create`, `upsert`, `delete`, `updateMany`, `deleteMany` produce no audit | Handle every write operation |
| 7 | `args.select` / `include` change the result shape | The diff only sees the selected fields, so it is incomplete | Run the write without projection, then re-read with the caller's projection |
| 8 | `BigInt` values in the diff | `JSON` column serialization **throws** (`Do not know how to serialize a BigInt`) | Normalize to string |
| 9 | Sensitive fields logged as-is | `passwordHash`, tokens and PII end up in a table that is kept forever | Per-model redaction list |
| 10 | Comment says "write audit log asynchronously" but the code `await`s | Misleading | It should be synchronous-in-tx (strong) or go via an outbox (async) |
| 11 | `userId` from `x-user-id` header | Anyone can spoof the actor | Take it from the authenticated principal |
| 12 | Q&A suggests `lodash.difference` for deep diff | That is *array set difference*, not a deep diff | `util.isDeepStrictEqual`, `lodash.isEqual`, or `microdiff` / `jsondiffpatch` for path-level diffs |

### 4.2 Schema

```prisma
model AuditLog {
  id         BigInt   @id @default(autoincrement())
  entityName String
  entityId   String
  action     String   // CREATE | UPDATE | DELETE
  userId     String
  requestId  String?
  source     String?  // http | job | script
  changes    Json     // { field: { old, new } }
  createdAt  DateTime @default(now())

  @@index([entityName, entityId, createdAt]) // "history of product 42"
  @@index([userId, createdAt])               // "what did user X do?"
}
```

### 4.3 Diff utility: `audit-diff.js`

```javascript
import { isDeepStrictEqual } from 'node:util';

const IGNORED_FIELDS = new Set(['updatedAt']);

/** Make values comparable AND JSON-serializable. */
function normalize(value) {
  if (value instanceof Date) return value.toISOString();
  if (typeof value === 'bigint') return value.toString();
  if (Buffer.isBuffer(value)) return value.toString('base64');
  if (value?.constructor?.name === 'Decimal') return value.toString(); // Prisma.Decimal
  return value ?? null;
}

/**
 * Diff two flat records (before = null for create, after = null for delete).
 * Returns { field: { old, new } } for fields that actually changed.
 */
export function diffRecords(before, after, redacted = new Set()) {
  const changes = {};
  const keys = new Set([...Object.keys(before ?? {}), ...Object.keys(after ?? {})]);

  for (const key of keys) {
    if (IGNORED_FIELDS.has(key)) continue;
    const oldVal = normalize(before?.[key]);
    const newVal = normalize(after?.[key]);
    if (isDeepStrictEqual(oldVal, newVal)) continue; // deep compare handles Json columns

    changes[key] = redacted.has(key)
      ? { old: '[REDACTED]', new: '[REDACTED]' } // record THAT it changed, not the value
      : { old: oldVal, new: newVal };
  }
  return changes;
}
```

### 4.4 The extension: `prisma.js`

```javascript
import { PrismaClient, Prisma } from '@prisma/client';
import { getAuditContext } from './context.js';
import { diffRecords } from './audit-diff.js';

// Explicit allow-list: audit the business entities only (never AuditLog itself).
const AUDITED_MODELS = new Set(['Product', 'Order', 'User']);
const REDACTED_FIELDS = { User: new Set(['passwordHash', 'mfaSecret']) };
const MAX_RETRIES = 3;

const delegateName = (model) => model[0].toLowerCase() + model.slice(1); // OrderItem -> orderItem

/** Run fn in a RepeatableRead tx; retry on write conflicts (Prisma P2034). */
async function inAuditTx(base, fn) {
  for (let attempt = 1; ; attempt++) {
    try {
      return await base.$transaction(fn, {
        isolationLevel: Prisma.TransactionIsolationLevel.RepeatableRead,
      });
    } catch (err) {
      if (err?.code === 'P2034' && attempt < MAX_RETRIES) continue;
      throw err;
    }
  }
}

function buildAuditRows(model, action, pairs) {
  const ctx = getAuditContext();
  const redacted = REDACTED_FIELDS[model];
  return pairs
    .map(({ before, after }) => ({
      entityName: model,
      entityId: String((after ?? before).id),
      action,
      userId: ctx.userId,
      requestId: ctx.requestId,
      source: ctx.source,
      changes: diffRecords(before, after, redacted),
    }))
    .filter((row) => Object.keys(row.changes).length > 0); // skip no-op updates
}

export function createAuditedClient() {
  const base = new PrismaClient(); // un-extended: used internally, so no recursion

  return base.$extends({
    name: 'audit',
    query: {
      $allModels: {
        async $allOperations({ model, operation, args, query }) {
          if (!AUDITED_MODELS.has(model)) return query(args);

          const d = delegateName(model);
          const { select, include, omit, ...writeArgs } = args ?? {};
          const project = (tx, id) =>
            select || include || omit ? tx[d].findUnique({ where: { id }, select, include, omit }) : null;

          switch (operation) {
            case 'create':
              return inAuditTx(base, async (tx) => {
                const after = await tx[d].create(writeArgs);
                await tx.auditLog.createMany({ data: buildAuditRows(model, 'CREATE', [{ before: null, after }]) });
                return (await project(tx, after.id)) ?? after;
              });

            case 'update':
            case 'upsert':
              return inAuditTx(base, async (tx) => {
                const before = await tx[d].findUnique({ where: writeArgs.where });
                const after = await tx[d][operation](writeArgs);
                const action = before ? 'UPDATE' : 'CREATE'; // upsert may create
                await tx.auditLog.createMany({ data: buildAuditRows(model, action, [{ before, after }]) });
                return (await project(tx, after.id)) ?? after;
              });

            case 'delete':
              return inAuditTx(base, async (tx) => {
                const before = await tx[d].findUnique({ where: writeArgs.where, select, include, omit });
                const deleted = await tx[d].delete({ where: writeArgs.where });
                await tx.auditLog.createMany({ data: buildAuditRows(model, 'DELETE', [{ before: deleted, after: null }]) });
                return before ?? deleted;
              });

            case 'updateMany':
            case 'deleteMany':
              // Bulk ops return only { count }, so snapshot rows by id before and after.
              return inAuditTx(base, async (tx) => {
                const befores = await tx[d].findMany({ where: args.where });
                const ids = befores.map((r) => r.id);
                const result = await tx[d][operation]({ ...args, where: { id: { in: ids } } });
                const afters = operation === 'updateMany'
                  ? new Map((await tx[d].findMany({ where: { id: { in: ids } } })).map((r) => [r.id, r]))
                  : new Map();
                const pairs = befores.map((before) => ({ before, after: afters.get(before.id) ?? null }));
                const action = operation === 'updateMany' ? 'UPDATE' : 'DELETE';
                await tx.auditLog.createMany({ data: buildAuditRows(model, action, pairs) });
                return result;
              });

            default: // reads, aggregates, createMany (see limitations)
              return query(args);
          }
        },
      },
    },
  });
}
```

**Why these choices (be ready to explain):**
- **`base` (un-extended) client inside the extension** → the audit's own writes don't re-enter the extension, so there is no infinite loop.
- **`RepeatableRead` + retry** → in Postgres, if another transaction commits a change to the row after our snapshot, our `UPDATE` fails with a serialization error (`P2034`) and we retry with a fresh snapshot. So the `old` values are never stale. An alternative is `SELECT … FOR UPDATE` via `$queryRaw`, but that needs per-model table names.
- **Bulk ops rewrite `where` to `id IN (…)`** → the rows we diff are exactly the rows we modify, even if new rows start matching the filter mid-flight.
- **Allow-list of models** → explicit, avoids auditing high-churn tables (sessions, metrics) and the audit table itself.

### 4.5 Known limitations of the extension approach (say these before the interviewer does)

| Limitation | Detail | Mitigation |
|---|---|---|
| **Nested writes** | `order.update({ data: { items: { update: … } } })` runs as one top-level operation, so the extension only sees `Order` and item changes are not diffed | Forbid nested writes on audited models (lint/review rule), or move to DB triggers |
| **Outer transactions** | If the caller is already in `db.$transaction(async tx => …)`, the extension opens a **separate** tx on `base`. The audit is then not part of the caller's tx, and it can **deadlock** on a row the caller already locked | Document it, use triggers for tx-heavy flows, or pass the tx explicitly to a small audit helper |
| **Raw SQL / other services / manual DB fixes** | `$executeRaw`, other microservices or `psql` bypass the app entirely | DB triggers (approach C) |
| **Extra round-trips** | Each update adds 1 read + 1 insert + tx overhead | Usually fine. For hot paths use triggers (done inside the DB, zero extra round-trips) |
| **`createMany`** | Doesn't return rows (`createManyAndReturn` exists on Postgres) | Use `createManyAndReturn` for audited models |
| **Bypasses other query extensions** | We call `tx[d].update` on the base client instead of `query(args)` | Put the audit extension last, or keep extensions composable |

### 4.6 The business service stays pure

```javascript
export class ProductService {
  constructor(db) { this.db = db; }

  updateProduct(id, data) {
    return this.db.product.update({ where: { id }, data }); // zero audit code
  }
}
```

### 4.7 Wiring (`server.js`)

```javascript
import express from 'express';
import { runWithAuditContext } from './context.js';
import { createAuditedClient } from './prisma.js';
import { ProductService } from './product.service.js';
import { authenticate } from './auth.js';

const app = express();
app.use(express.json());
app.use(authenticate);
app.use((req, res, next) =>
  runWithAuditContext({ userId: req.user.id, ip: req.ip, source: 'http' }, next),
);

const db = createAuditedClient();
const products = new ProductService(db);

app.patch('/products/:id', async (req, res, next) => {
  try {
    res.json(await products.updateProduct(Number(req.params.id), req.body)); // ids are Int
  } catch (err) {
    next(err);
  }
});

app.get('/products/:id/history', async (req, res) => {
  res.json(await db.auditLog.findMany({
    where: { entityName: 'Product', entityId: req.params.id },
    orderBy: { createdAt: 'desc' },
  }));
});

app.listen(3000);
```

### 4.8 Live demo script

```bash
curl -X PATCH localhost:3000/products/42 -H "Authorization: Bearer <alice>" \
     -H "Content-Type: application/json" -d '{"price": 80, "name": "Desk"}'

curl localhost:3000/products/42/history
```

```json
[{
  "entityName": "Product", "entityId": "42", "action": "UPDATE",
  "userId": "alice", "requestId": "3f1c…", "source": "http",
  "changes": { "price": { "old": "100", "new": "80" } },
  "createdAt": "2026-09-30T10:12:00.000Z"
}]
```

Note that `name` isn't in `changes`, because it didn't actually change (the no-op is filtered out). `price` is a string because it's a `Decimal`.

---

## 5. Approach C: Postgres trigger + session variable (strongest guarantee)

The original guide says triggers make it "extremely difficult to capture the userId". That isn't true. The standard technique is to **set a transaction-local config variable** from the app, and the trigger reads it.

### 5.1 SQL: generic trigger for any table

```sql
CREATE TABLE audit_log (
  id          bigserial   PRIMARY KEY,
  table_name  text        NOT NULL,
  entity_id   text        NOT NULL,
  action      text        NOT NULL,           -- INSERT | UPDATE | DELETE
  changed_by  text        NOT NULL,
  request_id  text,
  changes     jsonb       NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON audit_log (table_name, entity_id, created_at);

CREATE OR REPLACE FUNCTION audit_trigger() RETURNS trigger AS $$
DECLARE
  old_row jsonb := CASE WHEN TG_OP <> 'INSERT' THEN to_jsonb(OLD) END;
  new_row jsonb := CASE WHEN TG_OP <> 'DELETE' THEN to_jsonb(NEW) END;
  diff    jsonb;
BEGIN
  SELECT coalesce(jsonb_object_agg(key, jsonb_build_object('old', old_row -> key, 'new', new_row -> key)), '{}')
    INTO diff
    FROM jsonb_object_keys(coalesce(new_row, old_row)) AS key
   WHERE key NOT IN ('updatedAt')
     AND (old_row -> key) IS DISTINCT FROM (new_row -> key);

  IF diff = '{}' THEN RETURN NULL; END IF;   -- no-op UPDATE, nothing to log

  INSERT INTO audit_log (table_name, entity_id, action, changed_by, request_id, changes)
  VALUES (
    TG_TABLE_NAME,
    coalesce(new_row ->> 'id', old_row ->> 'id'),
    TG_OP,
    -- nullif: on pooled connections an unset custom GUC can read back as '' instead of NULL
    coalesce(nullif(current_setting('app.user_id', true), ''), 'db:' || current_user),
    nullif(current_setting('app.request_id', true), ''),
    diff
  );
  RETURN NULL;  -- AFTER trigger: return value is ignored
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER product_audit
AFTER INSERT OR UPDATE OR DELETE ON "Product"
FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

Put this in a Prisma migration (`prisma migrate dev --create-only`, then paste the SQL).

### 5.2 Node side: pass ALS context into the DB transaction

```javascript
import { PrismaClient } from '@prisma/client';
import { auditContext } from './context.js';

const WRITE_OPS = new Set(['create', 'createMany', 'update', 'updateMany', 'upsert', 'delete', 'deleteMany']);

export function createClientWithDbAudit() {
  const base = new PrismaClient();

  return base.$extends({
    name: 'audit-context',
    query: {
      $allModels: {
        async $allOperations({ operation, args, query }) {
          const ctx = auditContext.getStore();
          if (!ctx || !WRITE_OPS.has(operation)) return query(args);

          // set_config(..., true) = LOCAL to this transaction → safe with connection pooling.
          const [, result] = await base.$transaction([
            base.$executeRaw`SELECT set_config('app.user_id', ${ctx.userId}, true),
                                    set_config('app.request_id', ${ctx.requestId ?? ''}, true)`,
            query(args),
          ]);
          return result;
        },
      },
    },
  });
}
```

**Why it's strong:**
- ✅ Catches **everything**: raw SQL, nested writes, bulk ops, other services, migrations, manual `psql` fixes (attributed as `db:<role>`).
- ✅ Always atomic: the trigger runs inside the same transaction as the write.
- ✅ No extra network round-trips for the "before" read, because `OLD` is free inside the trigger.
- ❌ Logic lives in SQL (harder to unit test, versioned via migrations).
- ❌ Postgres-specific. Diff sees column names, not domain concepts.
- ❌ Adds write latency inside the transaction on very hot tables (measure it; consider statement-level triggers with transition tables for bulk ops).
- ⚠️ **Never use session-level `SET`** (non-local) with a pool: the next request on that connection would inherit the previous user.

---

## 6. Approach B: Repository / Decorator (ORM-agnostic)

This fits when there's no ORM (raw `pg`, Knex) or the team already uses repositories.

```javascript
import { getAuditContext } from './context.js';
import { diffRecords } from './audit-diff.js';

export class AuditedRepository {
  constructor(pool, table, { redacted = new Set() } = {}) {
    Object.assign(this, { pool, table, redacted });
  }

  async update(id, patch) {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      const { rows: [before] } = await client.query(
        `SELECT * FROM ${this.table} WHERE id = $1 FOR UPDATE`, [id]); // row lock: no stale "old"
      if (!before) throw new Error(`${this.table} ${id} not found`);

      const cols = Object.keys(patch);
      const set = cols.map((c, i) => `"${c}" = $${i + 2}`).join(', '); // cols must be whitelisted!
      const { rows: [after] } = await client.query(
        `UPDATE ${this.table} SET ${set} WHERE id = $1 RETURNING *`, [id, ...Object.values(patch)]);

      const changes = diffRecords(before, after, this.redacted);
      if (Object.keys(changes).length) {
        const { userId, requestId } = getAuditContext();
        await client.query(
          `INSERT INTO audit_log (table_name, entity_id, action, changed_by, request_id, changes)
           VALUES ($1, $2, 'UPDATE', $3, $4, $5)`,
          [this.table, String(id), userId, requestId, changes]);
      }
      await client.query('COMMIT');
      return after;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
}
```

**Trade-off:** it's explicit and easy to reason about (`FOR UPDATE` is simpler than isolation-level retries), but it only works if *every* write goes through the repository. Enforce that with an ESLint `no-restricted-imports` rule on the raw pool.

---

## 7. Approach D: Transactional Outbox / Domain Events (audit as a *business* fact)

Use this when the audit trail also feeds other consumers (search index, analytics, notifications, a separate audit service), or when **why** matters as much as **what**.

```javascript
// Service emits a meaningful event in the SAME transaction as the state change.
async applyDiscount(productId, percent, campaignId) {
  return this.db.$transaction(async (tx) => {
    const before = await tx.product.findUniqueOrThrow({ where: { id: productId } });
    const newPrice = before.price.mul(1 - percent / 100);
    const after = await tx.product.update({ where: { id: productId }, data: { price: newPrice } });

    await tx.outbox.create({
      data: {
        aggregate: 'Product',
        aggregateId: String(productId),
        type: 'ProductDiscountApplied',
        payload: { from: before.price.toString(), to: newPrice.toString(), percent, campaignId,
                   ...getAuditContext() },
      },
    });
    return after;
  });
}
```

```javascript
// Relay: polls the outbox and publishes to Kafka/RabbitMQ (at-least-once).
async function relayOutbox() {
  const batch = await db.outbox.findMany({ where: { publishedAt: null }, orderBy: { id: 'asc' }, take: 100 });
  for (const evt of batch) {
    await producer.send({ topic: 'audit.events', messages: [{ key: evt.aggregateId, value: JSON.stringify(evt) }] });
    await db.outbox.update({ where: { id: evt.id }, data: { publishedAt: new Date() } });
  }
}
// Consumers must be idempotent (dedupe on evt.id). Keying by aggregateId keeps per-entity ordering.
```

**Trade-off:** it captures intent (`campaignId`) and avoids dual-write problems, but it's explicit (the developer must emit the event) and the audit store is **eventually consistent**. This is often paired with approach A or C as a safety net.

---

## 8. Approach E: CDC with Debezium (platform-level)

- Debezium reads the Postgres **WAL** (logical replication) and publishes every row change to Kafka with `before` and `after` images (needs `REPLICA IDENTITY FULL` for full `before`).
- **Zero app code, zero write-path latency**, and it catches every writer.
- **The WAL has no idea who the user was.** Fix: add a `modified_by` / `request_id` column that the app sets on every write (for example, via the Prisma extension), so it appears in the `after` image. Deletes need a soft-delete or an outbox row to carry the actor.
- Makes sense when you **already run Kafka** and many systems consume changes. It's overkill just for an audit table.

---

## 9. Bonus: Mongoose plugin (if the interview is Mongo-based)

```javascript
import { diffRecords } from './audit-diff.js';
import { getAuditContext } from './context.js';
import { AuditLog } from './audit-log.model.js';

export function auditPlugin(schema, { modelName, redacted = new Set() }) {
  // 1) Document path: doc.save()
  schema.post('init', function () { this.$locals.original = this.toObject(); });

  schema.pre('save', function () {
    this.$locals.wasNew = this.isNew;
    this.$locals.changes = diffRecords(this.$locals.original ?? null, this.toObject(), redacted);
  });

  schema.post('save', async function (doc) {
    if (!Object.keys(doc.$locals.changes).length) return;
    await AuditLog.create({ entityName: modelName, entityId: String(doc._id),
      action: doc.$locals.wasNew ? 'CREATE' : 'UPDATE', changes: doc.$locals.changes, ...getAuditContext() });
  });

  // 2) Query path: Model.findOneAndUpdate() — `this` is the Query
  schema.pre('findOneAndUpdate', async function () {
    this._auditBefore = await this.model.findOne(this.getQuery()).lean();
  });

  schema.post('findOneAndUpdate', async function () {
    if (!this._auditBefore) return;
    const after = await this.model.findById(this._auditBefore._id).lean(); // result may be the pre-update doc
    const changes = diffRecords(this._auditBefore, after, redacted);
    if (Object.keys(changes).length)
      await AuditLog.create({ entityName: modelName, entityId: String(after._id),
        action: 'UPDATE', changes, ...getAuditContext() });
  });
}

// productSchema.plugin(auditPlugin, { modelName: 'Product' });
```

Mongoose caveats: `updateMany` / `updateOne` / `bulkWrite` need their own hooks. Atomicity needs a session/transaction (replica set). An alternative is **MongoDB Change Streams**, which have the same "who?" problem as CDC, fixed the same way (`modifiedBy` field).

---

## 10. Storage & operations concerns

| Concern | Pragmatic answer |
|---|---|
| **Diff vs. full snapshot** | A diff is compact and readable. A snapshot (`before` + `after`) makes "state at time T" trivial and is robust to schema changes. Common choice: store the diff, plus a periodic or on-delete snapshot |
| **Immutability** | App DB role gets `INSERT, SELECT` only on `audit_log`: `REVOKE UPDATE, DELETE ON audit_log FROM app_user;` |
| **Tamper-evidence** | Hash chain: `hash = sha256(prevHash + canonicalJSON(row))`. Or ship to WORM storage (S3 Object Lock) |
| **Volume / retention** | Partition by month (`PARTITION BY RANGE (created_at)`), drop or archive old partitions to cold storage (S3 + Athena/BigQuery) |
| **GDPR vs. immutable log** | Store user **IDs** instead of names/emails, redact PII fields, or use crypto-shredding (encrypt per-subject, delete the key to "forget") |
| **Query patterns** | Index `(entity, entity_id, created_at)` for entity history and `(user_id, created_at)` for user activity. Use a GIN index on `changes` only if you query inside the JSON |
| **Separate DB?** | Same DB gives atomicity. A separate store (Elastic, ClickHouse) gives cheap search and analytics. Get both with outbox/CDC → separate store |
| **Schema evolution** | Diffs store column names at the time of the change. Renamed columns break naive "replay" logic, so keep a `schema_version` if replay matters |

---

## 11. Decision guide

```text
Need to catch writes from raw SQL / other services / DBAs?     → DB triggers (C) or CDC (E)
Single Node service, Prisma, want the cleanest demo?            → Prisma extension (A), fixed version
Audit must also drive other systems / capture business intent?  → Outbox + domain events (D)
No ORM, raw pg/Knex?                                            → Repository (B) or triggers (C)
Already running Kafka + Debezium?                               → CDC (E) + modified_by column
Compliance-grade?                                               → C (atomic, un-bypassable) + append-only + retention policy
```

**My default recommendation:** **ALS for context + Postgres trigger with `set_config`** for the guaranteed row-level trail, plus **domain events via outbox** for flows where business intent matters. The Prisma extension is the best *whiteboard/demo* answer and is fine for a single-service app. Be upfront about its limits (nested writes, outer transactions, raw SQL).

---

## 12. Interview Q&A bank

**Q: What if the audit write fails? Should it block the update?**
It depends on the requirement. For **compliance**, yes: write it in the same transaction, so a failed audit rolls back the change ("no audit, no change"). For **best-effort activity feeds**, write asynchronously (outbox → queue) and never fail the user request. The original's `.catch(console.error)` lands in between: the data changes with no audit record and nobody notices.

**Q: How do you get the old values reliably under concurrency?**
There are three options. (1) Read the row `FOR UPDATE` in the same tx. (2) Use `RepeatableRead`/`Serializable` and retry on serialization failure. (3) Use a trigger, where `OLD` is exact by construction. A read outside the transaction gives a race.

**Q: Why `AsyncLocalStorage` instead of passing `userId` around?**
It keeps signatures clean across controller → service → repository → ORM, and it works for code you don't own (the ORM extension). The trade-off is implicit data flow: context can be lost across some callback-based libraries, so test it and provide a safe default (`system`).

**Q: How do you audit bulk updates (`updateMany`)?**
Snapshot the matching rows by id before and after inside one tx, then restrict the write to `id IN (…)` (section 4.4). For very large batches, use a statement-level trigger with `REFERENCING OLD TABLE AS o NEW TABLE AS n`, or log a single "bulk" audit event with the filter and count.

**Q: Deep/nested objects?**
Use `util.isDeepStrictEqual` for equality. If you need *path-level* diffs (`address.city`), use `microdiff` or `jsondiffpatch`. **Not** `lodash.difference`, which does array set difference.

**Q: How would you show "what did product 42 look like last Tuesday?"**
Either store full snapshots, or take the current state and **reverse-apply** diffs newer than T. If this is a core feature (not just an audit), consider temporal tables or event sourcing.

**Q: How do you stop the audit table from being modified?**
Give the app role `INSERT`/`SELECT` only. Add a hash chain for tamper-evidence, and export to WORM storage for regulated environments.

**Q: How does this scale to billions of rows?**
Use monthly partitions, move cold partitions to object storage, and keep only hot data in Postgres. Or stream via outbox/CDC into ClickHouse/Elastic for search and keep Postgres as short-term storage.

**Q: What about reads? Do we audit those too?**
Row diffs don't cover reads. Access auditing (HIPAA "who viewed this record") is a separate concern, usually handled at the API layer (middleware logging `userId + resource + requestId` to an append-only sink) or with `pgaudit` for statement-level DB logging.
