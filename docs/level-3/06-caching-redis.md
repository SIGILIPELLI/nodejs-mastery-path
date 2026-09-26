---
description: "Cache a Node API with Redis: the node-redis client, cache-aside with TTLs, invalidation, stampede protection, key design, and what not to cache."
---

# 06 · Caching with Redis

Some data is read far more often than it changes: a product page, a user's permissions,
an exchange-rate table, the result of an expensive aggregation. Caching keeps a copy
somewhere faster than the source of truth. For a Node service there are two common
places:

- **In-process memory** (a `Map` or an LRU library): nanoseconds to read, but each
  process has its own copy (lesson 05), it's lost on restart, and it competes with your
  heap.
- **Redis** (or a compatible server such as Valkey): an in-memory data store over the
  network. Sub-millisecond on a local network, shared by all processes, survives app
  restarts, and supports TTLs natively.

Redis is also used for sessions, rate limiting, queues (Level 4), pub/sub (lesson 07), and
distributed locks — so it's worth knowing well.

## Connecting with node-redis

```bash
npm install redis
```

```js title="src/redis.js"
import { createClient } from 'redis';

export async function connectRedis(url) {
  const client = createClient({ url });          // e.g. redis://localhost:6379
  client.on('error', (err) => console.error('redis error', err));   // required: see below
  await client.connect();
  return client;
}

// Basic commands
await client.set('greeting', 'hello', { expiration: { type: 'EX', value: 60 } }); // 60 s TTL
await client.get('greeting');     // 'hello' (or null when missing/expired)
await client.del('greeting');
await client.incr('page:views');  // atomic counter
```

You need a running Redis to try these (a local install, or `docker run -p 6379:6379
redis`). The code in this section was checked against the `redis` package's type
definitions (v6 at the time of writing, which marks the older `{ EX: 60 }` option style as
deprecated in favor of `expiration`), but **the Redis calls themselves were not run for
this lesson**. The cache logic below *was* run, against an in-memory stand-in with the
same `get`/`set`/`del` shape. `ioredis` is a popular alternative client with a similar API.

The `'error'` listener is not optional: the client is an `EventEmitter`, and an unhandled
`'error'` (Level 1, lesson 07) would crash your process on a Redis blip. The client
reconnects automatically; commands issued while disconnected are queued or rejected
depending on configuration.

## Cache-aside

The standard pattern: on read, try the cache; on a miss, load from the database and
store the result with a TTL. On write, update the database and **delete** the cache key.

```js title="src/cache.js"
// Cache-aside helper. `store` is a node-redis client (or anything with get/set/del).
export function createCache(store, { prefix = 'cache:', defaultTtl = 60 } = {}) {
  const inFlight = new Map();   // key -> promise, per process: collapses concurrent misses

  async function getOrLoad(key, loader, ttl = defaultTtl) {
    const fullKey = prefix + key;
    const hit = await store.get(fullKey);
    if (hit !== null) return JSON.parse(hit);

    if (inFlight.has(fullKey)) return inFlight.get(fullKey);
    const promise = (async () => {
      try {
        const value = await loader();
        const jitter = Math.floor(Math.random() * ttl * 0.1);   // spread expirations
        await store.set(fullKey, JSON.stringify(value), { expiration: { type: 'EX', value: ttl + jitter } });
        return value;
      } finally {
        inFlight.delete(fullKey);
      }
    })();
    inFlight.set(fullKey, promise);
    return promise;
  }

  const invalidate = (key) => store.del(prefix + key);
  return { getOrLoad, invalidate };
}
```

Using it in a route:

```js
router.get('/products/:id', validate({ params: idParams }), async (req, res) => {
  const { id } = req.valid.params;
  const product = await cache.getOrLoad(`product:${id}`, () => repo.findProduct(id), 300);
  if (!product) throw notFound();
  res.json(product);
});

router.patch('/products/:id', /* ... */ async (req, res) => {
  const updated = await repo.updateProduct(req.valid.params.id, req.valid.body);
  await cache.invalidate(`product:${updated.id}`);   // after the DB write succeeds
  res.json(updated);
});
```

## Worked example: stampede protection, observed

When a popular key expires, every concurrent request misses at once and hits the
database together — a **cache stampede**. The `inFlight` map collapses concurrent misses
*within a process* into one load:

```js title="cache-demo.js"
import { createCache } from './cache.js';
import { fakeRedis } from './fake-redis.js';
import { setTimeout as sleep } from 'node:timers/promises';

const store = fakeRedis();
const cache = createCache(store, { defaultTtl: 30 });

let dbQueries = 0;
async function loadProductFromDb(id) {
  dbQueries++;
  await sleep(50);                       // pretend this is a slow query
  return { id, name: 'Mechanical keyboard', priceCents: 8900 };
}

// 20 concurrent requests for the same cold key
const results = await Promise.all(
  Array.from({ length: 20 }, () => cache.getOrLoad('product:42', () => loadProductFromDb(42))),
);
console.log('concurrent requests:', results.length, '| db queries:', dbQueries);

await cache.getOrLoad('product:42', () => loadProductFromDb(42));
console.log('after a warm read   | db queries:', dbQueries);

await cache.invalidate('product:42');    // e.g. after UPDATE products ...
await cache.getOrLoad('product:42', () => loadProductFromDb(42));
console.log('after invalidation  | db queries:', dbQueries);
console.log(store.calls.filter(c => !c.startsWith('GET')));
```

```text
concurrent requests: 20 | db queries: 1
after a warm read   | db queries: 1
after invalidation  | db queries: 2
[
  'SET cache:product:42 EX 30',
  'DEL cache:product:42',
  'SET cache:product:42 EX 31'
]
```

Twenty simultaneous requests produced one database query. The second `SET` shows the TTL
**jitter** (31 instead of 30): if thousands of keys are cached at the same moment (say,
after a deploy), random jitter keeps them from all expiring in the same second.

Across *many* processes, each process may still load once. For very hot keys, add a
short Redis lock (`SET lock:key 1 NX` with a small expiry) so only one process refreshes,
or refresh proactively before expiry.

## What to cache, and what not to

Good candidates: read-heavy, tolerant of slight staleness, expensive to compute — product
details, public profiles, configuration, rendered fragments, results of external API
calls with rate limits.

Be careful with:

- **Per-user authorization decisions** — a stale "is admin" can outlive a revocation.
  Keep TTLs short or invalidate on change.
- **Anything where staleness costs money** — balances, inventory at checkout. Read the
  source of truth.
- **Null results** — caching "not found" (briefly) protects the database from repeated
  lookups of non-existent ids, but remember to invalidate on create.

Key design: namespace keys (`cache:product:42`), include a version when the cached shape
changes (`cache:v2:product:42`) so a deploy doesn't read old shapes, and never build keys
from unbounded user input without normalizing it.

## How It Actually Works

Redis is a single-threaded (for command execution) in-memory server. Every command
operates on data structures in RAM and is **atomic**: `INCR` from two clients can't
interleave. That's what makes it good for counters, locks, and rate limits.

The client speaks **RESP**, a simple text-framed protocol, over one TCP connection.
node-redis **pipelines** automatically: if your code issues many commands without
awaiting in between, they are written to the socket back to back and replies are matched
to requests in order. So one connection can serve all concurrent requests of a Node
process — unlike Postgres, you typically don't need a pool (except for blocking commands
such as `BLPOP` or for pub/sub subscribers, which occupy a connection).

**Expiration**: Redis stores an expiry timestamp per key. Expired keys are removed
lazily (when accessed) and by a periodic sampling process, so an expired key never
appears in results even if its memory is reclaimed a bit later. When memory hits
`maxmemory`, the configured **eviction policy** (e.g. `allkeys-lru`) decides what to drop —
set one for cache instances, or Redis will start refusing writes.

**Why delete instead of update on write?** If two writers race — A writes DB, B writes
DB, B sets cache, A sets cache — the cache ends up with A's stale value indefinitely.
Deleting means the next reader loads the current DB value. A narrow race remains (a
reader loading old data just before the delete), which the TTL bounds.

## Common mistakes

- **No `'error'` listener** on the Redis client.
- **Cache without TTLs** — stale data forever and unbounded memory.
- **Updating the cache before the DB write commits** (or when the transaction rolls back).
- **Caching huge objects** — serializing a 5 MB JSON blob on every hit blocks the event
  loop. Cache smaller pieces.
- **Treating the cache as durable** — Redis caches may be flushed or evicted at any time;
  the app must work (slower) without it.
- **Using `KEYS *`** in production to find keys — it blocks Redis while scanning
  everything. Use `SCAN`, or better, design keys so you don't need to search.

## Exercise

1. Add a `cache.getOrLoad` around the Level 2 `GET /tasks/:id` and invalidate on `PATCH`
   and `DELETE`. Write a test using the fake store proving a second read doesn't hit the DB.
2. Implement **negative caching**: when the loader returns `null`, cache a sentinel for
   10 seconds.
3. Write a fixed-window rate limiter with Redis `INCR` + `EXPIRE` (`key = rl:<ip>:<minute>`),
   and explain what happens at the window boundary.
4. If you can run Redis locally, connect with `redis-cli MONITOR` in another terminal and
   watch the commands your app sends. Compare with the fake store's `calls` list.
