---
description: "Capstone: design, build, and ship a production-ready Node service end to end, with a spec, milestones, a tested lifecycle module, and a review rubric."
---

# 10 · Capstone — A Production-Ready Service

The earlier projects each exercised one level. The capstone asks you to put the whole
course together in one service you'd be comfortable handing to an on-call rotation. You
choose the domain; the requirements are about *how* it's built. Expect this to take
several focused days, not an afternoon — that's the point.

## The brief

Build **"Bookings"**: a service where users reserve time slots on shared resources
(meeting rooms, equipment, lab machines). It has enough real constraints to exercise
everything: concurrency (two people booking the same slot), background work
(confirmation emails, reminders), real-time updates (a live availability board), and
reporting (utilization per resource).

If you prefer another domain (a URL shortener with analytics, an inventory service, a
ticketing API), keep the same non-functional requirements.

### Functional requirements

1. Users register and log in (sessions or JWT + refresh — justify your choice).
2. Admins create resources with opening hours; users list resources and their availability
   for a date.
3. Users book, list, and cancel their bookings. **Two overlapping bookings for the same
   resource must be impossible**, even under concurrent requests.
4. A confirmation email job is enqueued when a booking is created (and in the same
   transaction); a reminder job runs 30 minutes before the start.
5. A WebSocket (or SSE) feed pushes availability changes for a resource to connected
   clients.
6. `GET /reports/utilization?from=&to=` computes utilization per resource without blocking
   the event loop, whatever the date range.

### Non-functional requirements

| Area | Requirement | Where you learned it |
|------|-------------|----------------------|
| Structure | Domain + use cases independent of Express and the DB; composition root | L4-04 |
| Types | TypeScript, `strict`, `tsc` in CI; Zod at every boundary | L4-03, L2-04 |
| Data | PostgreSQL via Kysely (or your justified alternative); versioned migrations; constraints enforce invariants | L2-06 |
| Errors | One error format; domain errors mapped at the edge; no internals leaked | L2-05 |
| Security | scrypt/argon2, rate-limited auth, helmet, ownership checks in queries, `npm audit` clean at high severity | L2-07, L3-09 |
| Tests | Use-case tests with fakes; HTTP integration tests against real Postgres (PGlite or container); a concurrency test for double booking | L2-08 |
| Observability | Pino logs with request/trace ids and redaction; OTel traces; RED + event-loop metrics | L2-09, L4-08 |
| Performance | A CPU profile of the report endpoint under load, and what you changed | L4-01 |
| Operations | Dockerfile (multi-stage, non-root), `/livez` + `/readyz`, graceful shutdown with deadline, config validated at startup | L4-07, L4-09, L1-09 |
| Workers | Separate worker process for jobs; idempotent handlers; dead-letter handling | L4-05 |

## A head start: the lifecycle module

Operational plumbing is where capstones usually cut corners, so here is a tested module
to build on. It mounts health endpoints, starts resources in order, and on shutdown drains
the HTTP server and stops resources in **reverse** order under a hard deadline:

```js title="src/lifecycle.js"
// Service lifecycle: health endpoints, ordered startup/shutdown, and a hard deadline.
// resources: [{ name, start?: async () => void, stop: async () => void, check?: async () => void }]
import { monitorEventLoopDelay } from 'node:perf_hooks';

export function createLifecycle({ logger, resources = [], shutdownTimeoutMs = 10_000, drainDelayMs = 0 }) {
  let state = 'starting';               // starting -> ready -> draining -> stopped
  const loopDelay = monitorEventLoopDelay({ resolution: 20 });
  loopDelay.enable();

  function mount(app) {
    app.get('/livez', (req, res) => res.json({ status: 'alive' }));
    app.get('/readyz', async (req, res) => {
      if (state !== 'ready') return res.status(503).json({ status: state });
      const failed = [];
      await Promise.all(resources.filter(r => r.check).map(async (r) => {
        try { await r.check(); } catch { failed.push(r.name); }
      }));
      if (failed.length) return res.status(503).json({ status: 'degraded', failed });
      res.json({ status: 'ready', eventLoopP99Ms: Number((loopDelay.percentile(99) / 1e6).toFixed(1)) });
    });
  }

  async function start(server, port) {
    for (const r of resources) {
      if (r.start) { await r.start(); logger.info({ resource: r.name }, 'resource started'); }
    }
    await new Promise((resolve) => server.listen(port, resolve));
    state = 'ready';
    logger.info({ port: server.address().port }, 'service ready');

    const stop = async (reason) => {
      if (state === 'draining' || state === 'stopped') return;
      state = 'draining';
      logger.info({ reason }, 'draining');
      setTimeout(() => { logger.error('shutdown deadline exceeded, forcing exit'); process.exit(1); }, shutdownTimeoutMs).unref();
      if (drainDelayMs) await new Promise((r) => setTimeout(r, drainDelayMs));
      await new Promise((resolve) => { server.close(resolve); server.closeIdleConnections(); });
      for (const r of [...resources].reverse()) {           // stop in reverse start order
        try { await r.stop(); logger.info({ resource: r.name }, 'resource stopped'); }
        catch (err) { logger.error({ err, resource: r.name }, 'resource failed to stop'); }
      }
      loopDelay.disable();
      state = 'stopped';
      logger.info('stopped');
    };
    return { stop, get state() { return state; } };
  }

  return { mount, start };
}
```

```js title="test/lifecycle.test.js"
import { test } from 'node:test';
import assert from 'node:assert/strict';
import express from 'express';
import { createServer } from 'node:http';
import pino from 'pino';
import { createLifecycle } from '../src/lifecycle.js';

test('ready after start, degraded when a check fails, resources stopped in reverse order', async () => {
  const events = [];
  let dbHealthy = true;
  const resources = [
    { name: 'db', start: async () => events.push('start db'), stop: async () => events.push('stop db'),
      check: async () => { if (!dbHealthy) throw new Error('down'); } },
    { name: 'queue', start: async () => events.push('start queue'), stop: async () => events.push('stop queue') },
  ];
  const lifecycle = createLifecycle({ logger: pino({ level: 'silent' }), resources });
  const app = express();
  lifecycle.mount(app);
  const server = createServer(app);
  const handle = await lifecycle.start(server, 0);
  const url = `http://localhost:${server.address().port}`;

  assert.equal((await fetch(`${url}/readyz`)).status, 200);
  dbHealthy = false;
  const degraded = await fetch(`${url}/readyz`);
  assert.equal(degraded.status, 503);
  assert.deepEqual((await degraded.json()).failed, ['db']);

  await handle.stop('test');
  assert.equal(handle.state, 'stopped');
  assert.deepEqual(events, ['start db', 'start queue', 'stop queue', 'stop db']);
});
```

```text
✔ ready after start, degraded when a check fails, resources stopped in reverse order
ℹ pass 1
ℹ fail 0
```

Wire it in your composition root:

```js title="src/main.js (sketch)"
const lifecycle = createLifecycle({
  logger,
  drainDelayMs: config.DRAIN_DELAY_MS,
  resources: [
    { name: 'db', stop: () => db.destroy(), check: () => sql`select 1`.execute(db) },
    { name: 'telemetry', stop: () => otelSdk.shutdown() },
  ],
});
const app = createHttpAdapter({ useCases, logger });
lifecycle.mount(app);
const handle = await lifecycle.start(createServer(app), config.PORT);
for (const sig of ['SIGTERM', 'SIGINT']) process.once(sig, () => handle.stop(sig));
```

## Milestones

1. **Skeleton (day 1).** Repo, TypeScript config, lint, `node:test`, CI running typecheck
   + tests, config module, logger, lifecycle, `/livez` + `/readyz`, Dockerfile.
2. **Core domain (day 2).** Resources, availability calculation, booking rules as pure
   functions with thorough unit tests (edge cases: touching intervals, opening hours, time
   zones — store UTC, convert at the edges).
3. **Persistence (day 3).** Migrations and repositories. Enforce "no overlaps" in the
   database, not only in code. In PostgreSQL an **exclusion constraint** does this:
   `EXCLUDE USING gist (resource_id WITH =, tstzrange(starts_at, ends_at) WITH &&)` (needs
   the `btree_gist` extension). Write a test that fires 20 concurrent bookings for the same
   slot and asserts exactly one succeeds. (Checked while writing this lesson using PGlite
   with its `btree_gist` extension: an overlapping insert failed with
   `23P01 conflicting key value violates exclusion constraint`, while a booking starting
   exactly when the previous one ends was accepted, because `tstzrange` defaults to
   half-open `[start, end)` bounds.)
4. **HTTP + auth (day 4).** Routes, validation, auth, rate limiting, error mapping,
   integration tests.
5. **Async work (day 5).** Outbox or same-transaction job enqueue, worker process, email
   (log it, or use a local SMTP catcher such as Mailpit), reminders, dead letters.
6. **Real-time + reports (day 6).** Availability feed; utilization report computed in SQL
   or in a worker thread; profile it and record before/after numbers.
7. **Hardening (day 7).** OTel traces across API → worker; dashboards or at least a
   `/metrics` endpoint; a load test; a graceful-shutdown test; a README runbook.

## Deliverables

- The repository, with a README covering: how to run locally, architecture diagram,
  decisions and trade-offs (sessions vs JWT, queue choice, how double booking is
  prevented), and known limitations.
- CI green: typecheck, tests, audit.
- A short **runbook**: what each alert means and how to respond; how to deploy and roll
  back; how to run migrations safely.
- A profiling note: the report endpoint's profile before and after optimization, with your
  measured numbers.

## Review rubric (100 points)

| Area | Points | Full marks when… |
|------|-------:|-------------------|
| Correctness | 20 | All functional requirements work; double booking impossible under concurrency (tested) |
| Architecture | 15 | Core independent of frameworks; clear composition root; no business logic in routes |
| Tests | 15 | Meaningful unit + integration tests; failure paths covered; suite is fast and deterministic |
| Security | 15 | Hashing, rate limits, authorization in queries, validated input, no secrets in repo/image |
| Operability | 15 | Health checks, graceful shutdown (tested), config validation, structured logs with ids, traces |
| Performance | 10 | Report endpoint doesn't block the loop; profile-driven improvement documented |
| Documentation | 10 | README, decisions, runbook are accurate and concise |

Grade yourself honestly, then ask a peer to try breaking the service: concurrent
bookings, malformed input, killing the worker mid-job, stopping Postgres, sending SIGTERM
under load.

## How It Actually Works

The capstone's hardest guarantee — no overlapping bookings — is a good illustration of
where correctness really lives. A check in application code ("query for overlaps, then
insert") has a race: two requests can both see "no overlap" and both insert, because the
event loop interleaves them at every `await`, and multiple processes run truly in
parallel. Only the database, which serializes conflicting writes, can enforce it. An
exclusion constraint makes Postgres check each new row against existing rows using a GiST
index over the time ranges; the second concurrent insert fails with SQLSTATE `23P01`
(`exclusion_violation`), which your repository translates into a domain error and your
HTTP adapter into a 409.

The same pattern — push invariants to the component that can actually enforce them
atomically — runs through the course: unique constraints for emails, `SKIP LOCKED` for
job claiming, Redis `INCR` for counters, idempotency keys for retried requests.

## Common mistakes

- Starting with infrastructure polish before the domain works.
- Enforcing invariants only in application code.
- Tests that pass alone but fail in parallel because they share rows.
- Mocking the database in integration tests.
- Treating shutdown, timeouts, and observability as "later" — they shape the code.
- Undocumented decisions; a reviewer should understand *why*, not just *what*.

## Exercise

Complete the capstone. When you finish, write a one-page retrospective: what broke first
under load, which lesson's technique saved the most time, and what you would change if the
service had to handle ten times the traffic. Then compare your design with the reasoning in
the [System Design Mastery Path](https://sigilipelli.github.io/system-design-mastery-path/)
— which of its patterns did you end up needing?
