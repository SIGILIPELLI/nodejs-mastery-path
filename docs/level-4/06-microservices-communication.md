---
description: "How Node microservices talk: HTTP with fetch, timeouts, retries with jitter, circuit breakers, gRPC, and asynchronous events through a message broker."
---

# 06 · Microservices Communication

Once a system is split into several services, every function call that used to be
in-process becomes a **network call** — which can be slow, fail halfway, time out after
the other side already did the work, or hit a service that's mid-deploy. How services
communicate determines much of a system's reliability. This lesson covers the two
families — **synchronous request/response** (HTTP, gRPC) and **asynchronous messaging**
(events through a broker) — and the defensive client code Node services need.

A word of caution first: microservices trade in-process simplicity for independent
deployability and scaling. Many teams are better served by a well-structured modular
monolith (lesson 04) until organizational or scaling pressure makes the split worth it.

## Synchronous: HTTP with fetch

Node's global `fetch` (built on the **undici** HTTP client) is the default choice for
service-to-service HTTP. The raw call is one line; the production version needs:

- a **timeout** on every call (a hung dependency must not hang you),
- **bounded retries** only for transient failures, with **exponential backoff and jitter**,
- a **circuit breaker** so a dead dependency fails fast instead of consuming resources,
- propagated **request/trace ids** (lesson 08).

```js title="http-client.js"
import { setTimeout as sleep } from 'node:timers/promises';

export class CircuitOpenError extends Error {}

// Minimal circuit breaker: after `threshold` consecutive failures, fail fast for `coolDownMs`
export function circuitBreaker({ threshold = 5, coolDownMs = 10_000 } = {}) {
  let failures = 0;
  let openedAt = 0;
  return {
    async call(fn) {
      if (failures >= threshold) {
        if (Date.now() - openedAt < coolDownMs) throw new CircuitOpenError('circuit open');
        failures = threshold - 1;   // half-open: allow one trial request
      }
      try {
        const result = await fn();
        failures = 0;
        return result;
      } catch (err) {
        if (++failures >= threshold) openedAt = Date.now();
        throw err;
      }
    },
    get state() { return failures >= threshold ? 'open' : 'closed'; },
  };
}

const RETRYABLE = new Set([502, 503, 504]);

// GET JSON with a per-attempt timeout, bounded retries with jittered backoff
export async function getJson(url, { timeoutMs = 1000, retries = 2, headers = {} } = {}) {
  for (let attempt = 0; ; attempt++) {
    try {
      const res = await fetch(url, { headers, signal: AbortSignal.timeout(timeoutMs) });
      if (res.ok) return await res.json();
      if (!RETRYABLE.has(res.status) || attempt >= retries) {
        throw Object.assign(new Error(`upstream ${res.status}`), { status: res.status });
      }
    } catch (err) {
      const transient = err.name === 'TimeoutError' || err.cause?.code === 'ECONNREFUSED' || err.cause?.code === 'ECONNRESET';
      if (err.status || !transient || attempt >= retries) throw err;
    }
    const backoff = 100 * 2 ** attempt;
    await sleep(backoff / 2 + Math.random() * backoff / 2);   // "equal jitter"
  }
}
```

## Worked example: flaky, slow, and down dependencies

```js title="demo.js"
import { createServer } from 'node:http';
import { once } from 'node:events';
import { getJson, circuitBreaker, CircuitOpenError } from './http-client.js';

// A flaky "inventory" service: fails the first 2 calls with 503, then answers
let calls = 0;
let mode = 'flaky';
const inventory = createServer((req, res) => {
  calls++;
  if (mode === 'down' || (mode === 'flaky' && calls <= 2)) { res.writeHead(503).end(); return; }
  if (mode === 'slow') { setTimeout(() => res.end('{}'), 2000); return; }
  res.writeHead(200, { 'content-type': 'application/json' }).end(JSON.stringify({ sku: 'LAMP', inStock: 12 }));
});
inventory.listen(0);
await once(inventory, 'listening');
const url = `http://localhost:${inventory.address().port}/stock/LAMP`;

console.log('flaky ->', await getJson(url), `(server saw ${calls} calls)`);

mode = 'slow';
const t0 = performance.now();
await getJson(url, { timeoutMs: 300, retries: 1 }).catch((e) => console.log('slow  ->', e.name, `after ${Math.round(performance.now() - t0)} ms`));

mode = 'down';
const breaker = circuitBreaker({ threshold: 3, coolDownMs: 5000 });
calls = 0;
for (let i = 1; i <= 5; i++) {
  try { await breaker.call(() => getJson(url, { retries: 0 })); }
  catch (e) { console.log(`down  -> call ${i}: ${e instanceof CircuitOpenError ? 'fast fail (circuit open)' : e.message}`); }
}
console.log(`server saw ${calls} calls while down; breaker is ${breaker.state}`);
inventory.closeAllConnections();
inventory.close();
```

```text
flaky -> { sku: 'LAMP', inStock: 12 } (server saw 3 calls)
slow  -> TimeoutError after 676 ms
down  -> call 1: upstream 503
down  -> call 2: upstream 503
down  -> call 3: upstream 503
down  -> call 4: fast fail (circuit open)
down  -> call 5: fast fail (circuit open)
server saw 3 calls while down; breaker is open
```

- **Flaky:** two 503s were retried transparently; the caller got data on the third attempt.
- **Slow:** each attempt was cut off at 300 ms; with one retry and backoff the caller got a
  `TimeoutError` after ~676 ms instead of waiting for a 2-second response (or forever).
- **Down:** after three consecutive failures the breaker opened, and calls 4 and 5 failed
  immediately without touching the network — protecting both the caller's latency and the
  struggling service from extra load.

Rules for retries:

- Retry only **idempotent** operations (GET, PUT, DELETE) or requests with an
  idempotency key (Level 2, lesson 03). Retrying a plain `POST /payments` may double-charge.
- Retry only **transient** failures: timeouts, connection resets, 502/503/504. Never 400s.
- **Budget** retries: with 3 layers of services each retrying 3 times, one user request
  can become 27 calls to the bottom service during an outage ("retry storm"). Retry at one
  layer, and keep total time within the caller's own deadline.
- **Jitter** spreads retries out so thousands of clients don't retry in lockstep.

Mature libraries exist for this (e.g. `cockatiel` for retry/circuit-breaker/timeout
policies; `opossum` for circuit breakers). The hand-rolled version above shows what they do.

## Synchronous: gRPC

**gRPC** uses HTTP/2 and Protocol Buffers: you define services and messages in `.proto`
files, and generate typed clients and servers. In Node, `@grpc/grpc-js` with
`@grpc/proto-loader` (or code generators) implements it. Benefits: a strict, versionable
contract, compact binary encoding, streaming RPCs, and deadlines propagated across calls.
Costs: harder to inspect with `curl`, browser clients need a proxy (gRPC-Web), and more
tooling. It's common for internal east-west traffic, with REST/JSON at the public edge.

## Asynchronous: events and messages

Instead of the order service *calling* the email, inventory, and analytics services, it
**publishes an event** — `order.placed` with the order id — to a broker, and each
interested service consumes it at its own pace:

```text
order-service ── publish "order.placed" ──> broker ──> inventory-service (reserve stock)
                                                  ├──> email-service (send receipt)
                                                  └──> analytics-service
```

Benefits: the publisher doesn't know or wait for consumers; a consumer being down delays
its work but doesn't fail the order; new consumers can be added without touching the
publisher. Costs: **eventual consistency** (the receipt arrives a moment later), harder
debugging across hops, and the need for idempotent consumers and schema evolution.

Common brokers and Node clients: RabbitMQ (`amqplib`), Apache Kafka (`kafkajs` and other
clients), NATS (`nats`), cloud queues like Amazon SQS/SNS (`@aws-sdk/client-sqs`), and
Redis Streams. Broker setup is outside what this lesson can run locally; the concepts
below apply to all of them.

Messaging essentials:

- **At-least-once delivery** is the norm — consumers must be idempotent (store processed
  message ids, or use natural idempotency such as upserts).
- **Acknowledge after processing**, not before, or a crash loses the message.
- **Ordering** is usually guaranteed only within a partition/key (e.g. all events for one
  order), not globally.
- **Publish reliably** with the *transactional outbox*: write the event to an `outbox`
  table in the same DB transaction as the business change, and have a relay publish from
  the table. Otherwise "saved the order but failed to publish" (or the reverse) happens.
- **Version event schemas**; consumers must tolerate added fields.

## How It Actually Works

`fetch` in Node is implemented by **undici**, which keeps a **connection pool per origin**
with HTTP/1.1 keep-alive. Reusing connections avoids a TCP (and TLS) handshake per call —
significant for chatty service-to-service traffic. `AbortSignal.timeout(ms)` creates a
signal backed by a timer; when it fires, undici aborts the request, destroys (or returns)
the socket, and the promise rejects with a `DOMException` named `TimeoutError`. Connection
failures surface as `TypeError: fetch failed` with the underlying system error in
`err.cause` — which is why the client checks `err.cause?.code`.

A **circuit breaker** is a small state machine: *closed* (calls pass, failures counted),
*open* (calls fail immediately until a cool-down passes), *half-open* (one trial call; if it
succeeds, close; if it fails, open again). It converts a slow failure (timeouts that tie up
sockets, memory, and event-loop capacity) into a fast one.

With a broker, the publisher writes a message to the broker over a persistent TCP
connection; the broker persists it (to disk or replicated memory) and pushes it to
subscribed consumers, tracking which messages each consumer group has acknowledged.
Unacknowledged messages are redelivered after a timeout or reconnect — the source of
at-least-once semantics.

## Common mistakes

- **No timeouts** on outbound calls.
- **Retrying non-idempotent requests**, or retrying at every layer.
- **Synchronous call chains** five services deep for one user request — latency and
  failure probabilities multiply.
- **Sharing a database between services**, coupling them more tightly than any API would.
- **Publishing events outside the DB transaction** without an outbox.
- **Consumers that assume exactly-once delivery.**

## Exercise

1. Add request-id propagation to `getJson`: accept an id and send it as `x-request-id`, and
   log it on the server.
2. The breaker above handles half-open implicitly (by resetting the failure count). Make
   `half-open` an explicit state that admits exactly one trial call even under concurrent
   requests, and write tests for each transition using an injected clock.
3. Write an outbox relay against PGlite: an `outbox` table filled in the same transaction
   as an `orders` insert, and a loop that "publishes" (logs) unpublished rows using
   `FOR UPDATE SKIP LOCKED` and marks them published.
4. Draw the communication for "user places an order" in an e-commerce system: which calls
   must be synchronous (the user is waiting for the answer) and which can be events?
