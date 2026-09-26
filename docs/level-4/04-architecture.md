---
description: "Structure maintainable Node services: layered architecture, hexagonal ports and adapters, dependency injection without a framework, and use-case tests."
---

# 04 · Architecture: Layered & Hexagonal

Express doesn't tell you where code goes. Small services survive that fine; at 30
endpoints and three developers, business rules end up scattered across route handlers,
SQL is duplicated, and changing the database or adding a queue consumer means touching
everything. Architecture is the set of decisions about **which code may depend on which**,
made so that the parts most likely to change (frameworks, databases, vendors) can change
without dragging the business rules with them.

## Layered architecture

The most common structure for Node APIs has three layers, each depending only on the one
below:

```text
routes / controllers   HTTP: parse & validate input, call a service, shape the response
        │
services               business rules and workflows ("place an order")
        │
repositories           data access: SQL/Kysely, Redis, external APIs
```

Rules of thumb:

- Route handlers contain no business decisions and no SQL — they translate HTTP.
- Services contain no `req`/`res` — they take plain arguments and return plain values, so
  a CLI, a queue worker, or a test can call them.
- Repositories expose intention-revealing methods (`findOpenTasksForUser`) rather than
  leaking the query builder everywhere.

This alone fixes most maintainability problems in typical services, and the Level 2
project could evolve into it by extracting `tasksService` from the routes.

## Hexagonal architecture (ports & adapters)

Layering still lets services depend *directly* on concrete infrastructure: a service
imports the Kysely repository or the Stripe client. **Hexagonal architecture** inverts
that: the core (domain + use cases) defines **ports** — the interfaces it needs — and the
outside world provides **adapters** that implement them.

```text
            driving adapters                         driven adapters
        (call into the core)                     (called by the core)

   HTTP (Express) ─┐                          ┌─> Postgres orders repository
   Queue consumer ─┼──>  use cases + domain ──┼─> Payment provider client
   CLI / cron     ─┘      (ports defined here) └─> Clock, id generator, email
```

The dependency arrows all point **inward**: the core imports nothing from Express,
Kysely, or an SDK. That makes the core fast to test (plug in in-memory adapters), and
makes infrastructure swappable.

## Worked example: placing an order

The domain: pure rules, no I/O.

```js title="src/domain/order.js"
// Domain: pure business rules. No Express, no SQL, no fetch.
export class DomainError extends Error {
  constructor(code, message) { super(message); this.code = code; }
}

export function createOrder({ id, customerId, lines, now }) {
  if (lines.length === 0) throw new DomainError('EMPTY_ORDER', 'an order needs at least one line');
  if (lines.length > 50) throw new DomainError('TOO_MANY_LINES', 'at most 50 lines per order');
  const totalMinor = lines.reduce((sum, l) => sum + l.unitPriceMinor * l.quantity, 0);
  return { id, customerId, lines, totalMinor, status: 'pending_payment', createdAt: now };
}

export function markPaid(order, paymentId) {
  if (order.status !== 'pending_payment') {
    throw new DomainError('INVALID_STATE', `cannot pay an order in status ${order.status}`);
  }
  return { ...order, status: 'paid', paymentId };
}
```

The use case receives its ports as arguments — dependency injection with nothing but
function parameters:

```js title="src/app/place-order.js"
// Application service (use case). Depends only on "ports": objects passed in.
//   orders:   { save(order), findById(id) }
//   catalog:  { priceOf(sku) -> number | null }
//   payments: { charge({ customerId, amountMinor, idempotencyKey }) -> { paymentId } }
//   clock:    { now() -> Date },  ids: { next() -> string }
import { createOrder, markPaid, DomainError } from '../domain/order.js';

export function makePlaceOrder({ orders, catalog, payments, clock, ids }) {
  return async function placeOrder({ customerId, items }) {
    const lines = [];
    for (const { sku, quantity } of items) {
      const unitPriceMinor = await catalog.priceOf(sku);
      if (unitPriceMinor == null) throw new DomainError('UNKNOWN_SKU', `unknown product ${sku}`);
      lines.push({ sku, quantity, unitPriceMinor });
    }
    const order = createOrder({ id: ids.next(), customerId, lines, now: clock.now() });
    await orders.save(order);

    const { paymentId } = await payments.charge({
      customerId, amountMinor: order.totalMinor, idempotencyKey: `order-${order.id}`,
    });
    const paid = markPaid(order, paymentId);
    await orders.save(paid);
    return paid;
  };
}
```

In-memory driven adapters for tests and local development:

```js title="src/adapters/memory.js"
// Driven adapters for tests and local development
export function memoryOrders() {
  const rows = new Map();
  return {
    async save(order) { rows.set(order.id, structuredClone(order)); },
    async findById(id) { return rows.has(id) ? structuredClone(rows.get(id)) : null; },
  };
}
export const fixedCatalog = (prices) => ({ async priceOf(sku) { return prices[sku] ?? null; } });
export const sequentialIds = () => { let n = 0; return { next: () => String(++n) }; };
export const fixedClock = (iso) => ({ now: () => new Date(iso) });
```

The HTTP driving adapter, the only module that imports Express:

```js title="src/adapters/http.js"
// Driving adapter: translates HTTP <-> use case. The only file that knows about Express.
import express from 'express';
import { z } from 'zod';
import { DomainError } from '../domain/order.js';

const Body = z.object({
  customerId: z.string().min(1),
  items: z.array(z.object({ sku: z.string().min(1), quantity: z.number().int().min(1).max(99) })).max(50),
});

const STATUS = { EMPTY_ORDER: 400, TOO_MANY_LINES: 400, UNKNOWN_SKU: 422, INVALID_STATE: 409 };

export function httpAdapter({ placeOrder }) {
  const app = express();
  app.use(express.json());
  app.post('/orders', async (req, res) => {
    const parsed = Body.safeParse(req.body);
    if (!parsed.success) return res.status(400).json({ error: 'validation failed' });
    const order = await placeOrder(parsed.data);
    res.status(201).json(order);
  });
  app.use((err, req, res, next) => {
    if (err instanceof DomainError) return res.status(STATUS[err.code] ?? 400).json({ error: err.message, code: err.code });
    res.status(500).json({ error: 'internal server error' });
  });
  return app;
}
```

Tests exercise the use case directly with fakes, plus one thin test of the HTTP mapping:

```js title="test/place-order.test.js"
import { test } from 'node:test';
import assert from 'node:assert/strict';
import request from 'supertest';
import { makePlaceOrder } from '../src/app/place-order.js';
import { memoryOrders, fixedCatalog, sequentialIds, fixedClock } from '../src/adapters/memory.js';
import { httpAdapter } from '../src/adapters/http.js';

function setup({ chargeFails = false } = {}) {
  const orders = memoryOrders();
  const charges = [];
  const payments = {
    async charge(req) {
      charges.push(req);
      if (chargeFails) throw new Error('card declined');
      return { paymentId: `pay_${charges.length}` };
    },
  };
  const placeOrder = makePlaceOrder({
    orders, payments, catalog: fixedCatalog({ LAMP: 2599, BULB: 350 }),
    clock: fixedClock('2025-03-01T10:00:00Z'), ids: sequentialIds(),
  });
  return { orders, charges, placeOrder };
}

test('computes the total, charges once with an idempotency key, and stores the paid order', async () => {
  const { orders, charges, placeOrder } = setup();
  const order = await placeOrder({ customerId: 'c1', items: [{ sku: 'LAMP', quantity: 1 }, { sku: 'BULB', quantity: 4 }] });
  assert.equal(order.totalMinor, 2599 + 4 * 350);
  assert.equal(order.status, 'paid');
  assert.deepEqual(charges, [{ customerId: 'c1', amountMinor: 3999, idempotencyKey: 'order-1' }]);
  assert.equal((await orders.findById('1')).status, 'paid');
});

test('a failed charge leaves the order pending, not paid', async () => {
  const { orders, placeOrder } = setup({ chargeFails: true });
  await assert.rejects(placeOrder({ customerId: 'c1', items: [{ sku: 'LAMP', quantity: 1 }] }), /declined/);
  assert.equal((await orders.findById('1')).status, 'pending_payment');
});

test('HTTP adapter maps domain errors to status codes', async () => {
  const app = httpAdapter(setup());
  const res = await request(app).post('/orders').send({ customerId: 'c1', items: [{ sku: 'NOPE', quantity: 1 }] });
  assert.equal(res.status, 422);
  assert.equal(res.body.code, 'UNKNOWN_SKU');
});
```

```text
✔ computes the total, charges once with an idempotency key, and stores the paid order
✔ a failed charge leaves the order pending, not paid
✔ HTTP adapter maps domain errors to status codes
ℹ pass 3
ℹ fail 0
```

The **composition root** — typically `src/main.js` — is the one place that knows about
everything and wires it together:

```js title="src/main.js (sketch)"
const db = createDb({ connectionString: config.DATABASE_URL });
const placeOrder = makePlaceOrder({
  orders: postgresOrders(db),           // implements save/findById with Kysely
  catalog: postgresCatalog(db),
  payments: paymentProviderClient(config.PAYMENTS_API_KEY),
  clock: { now: () => new Date() },
  ids: { next: () => crypto.randomUUID() },
});
httpAdapter({ placeOrder }).listen(config.PORT);
// A queue worker (lesson 05) could call the same placeOrder.
```

Notice what the tests *could* verify because time, ids, and payments are injected: exact
totals, the idempotency key sent to the payment provider, and the state left behind
when payment fails — a scenario that's awkward to reproduce against a real provider.

## How much architecture?

- A small CRUD service: routes + a thin repository is fine. Don't invent ports for
  things that will never change.
- Growing business rules, several entry points (HTTP + queue + cron), or external
  vendors you might replace: move to services and ports.
- Avoid ceremony for its own sake — an interface with exactly one implementation *and* no
  testing benefit is just indirection.

Folder structure should follow the architecture you chose. Many teams prefer grouping by
**feature** (`orders/`, `users/`, each with its routes, service, repository) over grouping
by technical layer, so that a feature's code lives together.

## How It Actually Works

JavaScript has no `interface` keyword at runtime, so "ports" in plain JS are just the
**shape** of the objects passed in (duck typing). The contract lives in documentation,
JSDoc, or TypeScript interfaces (lesson 03), and — most importantly — in the tests that
exercise both the in-memory and the real adapters against the same expectations
("contract tests").

Dependency injection here needs no container: `makePlaceOrder(deps)` returns a closure
that captures `deps`. Each call to the factory creates an independently wired instance,
which is why every test gets fresh state. Module-level singletons (`import db from
'./db.js'` inside a service) are the opposite: ES modules are cached per process, so
every importer shares one instance, and swapping it in a test requires module mocking.

The dependency rule is enforceable: tools like `eslint-plugin-boundaries` or
`dependency-cruiser` can fail the build when `src/domain` imports from `src/adapters`.

## Common mistakes

- **Business logic in route handlers** — untestable without HTTP, unreusable from workers.
- **Services that accept `req`** or return HTTP status codes.
- **Leaking ORM/query-builder types through the core**, so the "domain" can't exist
  without the database library.
- **Global singletons imported everywhere** instead of passing dependencies.
- **Over-engineering** a 5-endpoint service with layers of interfaces and factories.
- **Domain errors as HTTP errors** — the domain should say `UNKNOWN_SKU`; the HTTP adapter
  decides it's a 422.

## Exercise

1. Write `postgresOrders(db)` implementing `save` (upsert) and `findById` with Kysely, and
   run the same use-case tests against it using PGlite from Level 2.
2. Add a `cancelOrder` use case with the rule "only unpaid orders, or paid orders within
   30 minutes, can be cancelled". Inject the clock and test both boundaries.
3. Add a second driving adapter: a CLI (`node src/cli.js place-order --customer c1 --item
   LAMP:1`) that calls the same `placeOrder`.
4. Refactor the Level 2 tasks API into routes → service → repository and note which tests
   became easier to write.
