---
description: "Move slow work out of requests with job queues: a runnable Postgres SKIP LOCKED queue, BullMQ on Redis, retries, backoff, and idempotent jobs."
---

# 05 · Queues & Background Jobs

Some work doesn't belong inside an HTTP request: sending emails, generating PDFs,
resizing images, calling a slow partner API, recalculating reports, delivering webhooks.
Doing it inline makes responses slow, ties user-facing latency to third-party
reliability, and loses the work if the process restarts mid-request. The standard answer
is a **job queue**: the request records "this needs doing" and returns (often with
`202 Accepted`); separate **worker** processes do the work, with retries.

```text
 API process                    queue (DB / Redis / broker)           worker process(es)
 POST /signup ── enqueue ─────>  [ send-welcome-email #812 ]  ──claim──> sendEmail()
   └── 201 immediately                                                  ├─ success → done
                                                                        └─ failure → retry later / dead
```

## Options in the Node ecosystem

| Backend | Examples | Notes |
|---------|----------|-------|
| Redis | **BullMQ** | The most widely used Node job library: delays, retries, priorities, rate limits, repeatable jobs, dashboards |
| PostgreSQL | pg-boss, Graphile Worker, or hand-rolled | No extra infrastructure; jobs can be enqueued in the same transaction as your data |
| Message brokers | RabbitMQ, Amazon SQS, Kafka, NATS | For cross-service messaging and very high throughput (next lesson) |

## Worked example: a Postgres queue you can run

To see the mechanics without extra infrastructure, here is a compact queue on Postgres
using `FOR UPDATE SKIP LOCKED` — the same technique libraries like pg-boss and Graphile
Worker build on.

```js title="queue.js"
// A small Postgres-backed job queue. `sql` is a function (text, params) => rows.
export async function setupQueue(sql) {
  await sql(`
    create table if not exists jobs (
      id          bigserial primary key,
      queue       text not null,
      payload     jsonb not null,
      status      text not null default 'queued',   -- queued | running | done | dead
      attempts    int not null default 0,
      max_attempts int not null default 5,
      run_at      timestamptz not null default now(),
      last_error  text,
      locked_at   timestamptz
    )`);
  await sql(`create index if not exists jobs_ready_idx on jobs (queue, run_at) where status = 'queued'`);
}

export function enqueue(sql, queue, payload, { delaySeconds = 0, maxAttempts = 5 } = {}) {
  return sql(
    `insert into jobs (queue, payload, run_at, max_attempts)
     values ($1, $2, now() + make_interval(secs => $3), $4) returning id`,
    [queue, JSON.stringify(payload), delaySeconds, maxAttempts],
  );
}

// Atomically claim one ready job. SKIP LOCKED lets many workers poll without blocking each other.
async function claim(sql, queue) {
  const rows = await sql(
    `update jobs set status = 'running', attempts = attempts + 1, locked_at = now()
     where id = (
       select id from jobs
       where queue = $1 and status = 'queued' and run_at <= now()
       order by run_at, id
       for update skip locked
       limit 1
     )
     returning id, payload, attempts, max_attempts`,
    [queue],
  );
  return rows[0] ?? null;
}

export async function workOnce(sql, queue, handler, { baseBackoffSeconds = 2 } = {}) {
  const job = await claim(sql, queue);
  if (!job) return false;
  try {
    await handler(job.payload, job);
    await sql(`update jobs set status = 'done', locked_at = null where id = $1`, [job.id]);
  } catch (err) {
    const dead = job.attempts >= job.max_attempts;
    const backoff = baseBackoffSeconds * 2 ** (job.attempts - 1);   // 2s, 4s, 8s, ...
    await sql(
      `update jobs set status = $2, last_error = $3, locked_at = null,
              run_at = now() + make_interval(secs => $4)
       where id = $1`,
      [job.id, dead ? 'dead' : 'queued', String(err.message).slice(0, 500), backoff],
    );
  }
  return true;
}
```

A demo run against PGlite (in-process Postgres), with one good address and one that
always bounces:

```js title="demo.js"
import { PGlite } from '@electric-sql/pglite';
import { setupQueue, enqueue, workOnce } from './queue.js';

const pg = new PGlite();
const sql = async (text, params) => (await pg.query(text, params)).rows;
await setupQueue(sql);

await enqueue(sql, 'email', { to: 'ada@example.com', template: 'welcome' });
await enqueue(sql, 'email', { to: 'bounce@example.com', template: 'welcome' }, { maxAttempts: 3 });

let sent = 0;
async function sendEmail(payload) {
  if (payload.to.startsWith('bounce')) throw new Error('SMTP 550 mailbox unavailable');
  sent++;
}

// Drive the worker; use zero backoff so the demo doesn't wait for real seconds
for (let i = 0; i < 10; i++) {
  const didWork = await workOnce(sql, 'email', sendEmail, { baseBackoffSeconds: 0 });
  if (!didWork) break;
}
console.log('emails sent:', sent);
console.table(await sql(`select id, payload->>'to' as "to", status, attempts, last_error from jobs order by id`));
```

```text
emails sent: 1
┌─────────┬────┬──────────────────────┬────────┬──────────┬────────────────────────────────┐
│ (index) │ id │ to                   │ status │ attempts │ last_error                     │
├─────────┼────┼──────────────────────┼────────┼──────────┼────────────────────────────────┤
│ 0       │ 1  │ 'ada@example.com'    │ 'done' │ 1        │ null                           │
│ 1       │ 2  │ 'bounce@example.com' │ 'dead' │ 3        │ 'SMTP 550 mailbox unavailable' │
└─────────┴────┴──────────────────────┴────────┴──────────┴────────────────────────────────┘
```

The bouncing job was retried until `max_attempts` and then parked as `dead` with its last
error — a **dead-letter** state that a human or an alert can inspect, instead of retrying
forever. In production, the retry delays grow exponentially (2 s, 4 s, 8 s...), giving a
flaky dependency time to recover.

A real worker process loops: claim a job; if none, sleep briefly (or use Postgres
`LISTEN/NOTIFY` to wake up); handle `SIGTERM` by finishing the current job before exiting
(lesson 09). You'd also add a sweeper that re-queues jobs stuck in `running` with an old
`locked_at` — the trace of a worker that crashed mid-job.

**The big advantage of a database queue:** you can enqueue in the **same transaction** as
the business write. "Create user" and "enqueue welcome email" commit together or not at
all. With a separate broker you need the *transactional outbox* pattern to get the same
guarantee (see the [System Design Mastery Path](https://sigilipelli.github.io/system-design-mastery-path/)).

## The same idea with BullMQ

With Redis available, BullMQ gives you this and much more out of the box. The shape of the
code (this snippet needs a running Redis and was not executed for this lesson):

```js
import { Queue, Worker } from 'bullmq';
const connection = { host: 'localhost', port: 6379 };

// Producer (in the API)
const emails = new Queue('email', { connection });
await emails.add('welcome', { to: 'ada@example.com' }, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 2000 },
  jobId: `welcome:${userId}`,          // dedupe: the same id is not added twice
  removeOnComplete: 1000,
});

// Consumer (in a separate worker process)
const worker = new Worker('email', async (job) => {
  await sendEmail(job.data);
}, { connection, concurrency: 10 });
worker.on('failed', (job, err) => logger.warn({ jobId: job?.id, err }, 'job failed'));
```

## Designing jobs

- **Make handlers idempotent.** Queues deliver *at least once*: a worker can finish the
  work and crash before marking the job done, so the job runs again. Sending the same
  email twice is annoying; charging a card twice is serious. Use idempotency keys with
  external APIs, check "already done" state, or use unique constraints.
- **Put ids in payloads, not whole objects.** `{ orderId: 42 }`, then load fresh data in
  the worker. Payloads go stale; the database is the source of truth.
- **Keep jobs small and bounded.** One job per email, not one job for 50,000 emails.
- **Separate queues by priority/latency**, so a backlog of reports doesn't delay password
  reset emails.
- **Monitor** queue depth, age of the oldest job, failure rate, and dead jobs.
- **Scheduled/recurring work** (nightly cleanup) should be enqueued by exactly one
  scheduler, not a `setInterval` in every API replica.

## How It Actually Works

`SELECT ... FOR UPDATE` locks the selected row until the transaction ends. Normally a
second worker running the same query would **wait** for that lock. `SKIP LOCKED` tells
Postgres to skip rows another transaction has locked and take the next one instead, so N
workers claim N different jobs concurrently without blocking. Wrapping the select in an
`UPDATE ... WHERE id = (subquery) RETURNING` makes claim-and-mark-running a single atomic
statement. The partial index on `(queue, run_at) WHERE status = 'queued'` keeps the claim
query fast even when the table contains millions of finished jobs.

BullMQ stores each queue as a set of Redis structures — lists and sorted sets for waiting,
delayed, active, completed, and failed jobs, and a hash per job — and moves jobs between
them with **Lua scripts**, which Redis executes atomically. A worker blocks on Redis
(`BZPOPMIN`-style blocking commands) for new jobs rather than polling, holds a **lock**
with an expiry while processing, and renews it periodically; if the worker dies, the lock
expires and the job is detected as *stalled* and moved back to waiting.

## Common mistakes

- **Doing slow work in the request** "because it's usually fast".
- **Non-idempotent handlers** under at-least-once delivery.
- **Infinite retries** with no dead-letter state or alerting.
- **Enqueuing before the transaction commits** — the worker may run before the data
  exists, or for data that was rolled back.
- **Large payloads** (whole documents, images) in the queue.
- **Running workers inside the API process** so a burst of jobs slows down requests.
  Separate processes (or at least separate deployments) scale and fail independently.

## Exercise

1. Add a `runWorker(sql, queue, handler, { signal })` loop around `workOnce` that sleeps
   500 ms when idle and stops cleanly when the `AbortSignal` fires.
2. Start three workers concurrently against the same PGlite database (they will share one
   connection there; describe what would differ with a real Postgres pool) and enqueue 100
   jobs. Assert each job ran exactly once.
3. Add a sweeper that re-queues `running` jobs whose `locked_at` is older than 5 minutes,
   and a test that simulates a crashed worker.
4. In the Level 2 project, change registration to insert the user and enqueue a
   `welcome-email` job in the same Kysely transaction.
