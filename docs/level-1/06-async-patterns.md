---
description: "Node's async evolution from error-first callbacks to promises and async/await: promisify, parallel awaits, cancellation, and unhandled rejections."
---

# 06 · Async Patterns: Callbacks to async/await

This is the one lesson that overlaps with general JavaScript, because Node's async
history explains a lot of the APIs and code you will meet. If you already write
`async/await` comfortably, focus on the Node-specific parts: error-first callbacks,
`promisify`, cancellation with `AbortSignal`, and what happens to unhandled rejections.

## Stage 1: error-first callbacks

Node's original convention: the **last argument** is a callback, and the callback's
**first parameter** is an error (or `null`).

```js
import fs from 'node:fs';

fs.readFile('config.json', 'utf8', (err, text) => {
  if (err) {
    console.error('could not read config:', err.message);
    return;
  }
  const config = JSON.parse(text);
  console.log(config);
});
```

Callbacks work, but sequencing several steps nests them ("callback pyramid"), errors
must be checked manually at every level, and an exception thrown inside a callback
cannot be caught by a `try` around the original call — the stack has already unwound.

## Stage 2: promises

A promise represents a value that will exist later. Most Node built-ins now have
promise versions (`node:fs/promises`, `node:timers/promises`, `node:stream/promises`,
`node:dns/promises`). For callback-only APIs, `util.promisify` converts any function
that follows the error-first convention:

```js
import { promisify } from 'node:util';
import { execFile } from 'node:child_process';

const execFileP = promisify(execFile);
const { stdout } = await execFileP('git', ['rev-parse', 'HEAD']);
```

## Stage 3: async/await

`await` pauses the current `async` function until a promise settles, and throws on
rejection — so ordinary `try/catch` works again:

```js
import { readFile } from 'node:fs/promises';

async function loadConfig(file) {
  try {
    return JSON.parse(await readFile(file, 'utf8'));
  } catch (err) {
    if (err.code === 'ENOENT') return {};   // missing file → defaults
    throw new Error(`invalid config in ${file}`, { cause: err });
  }
}
```

Note `{ cause: err }`: wrapping errors while keeping the original preserves the root
cause for logs.

## Worked example: sequential vs parallel, partial failure, timeouts

```js title="async.mjs"
import { setTimeout as sleep } from 'node:timers/promises';

// Pretend these are network calls with different latencies
async function fetchUser(id) { await sleep(100); return { id, name: `user${id}` }; }
async function fetchOrders(id) { await sleep(150); return [{ id: 1, userId: id }]; }
async function fetchFlaky() { await sleep(50); throw new Error('recommendations service down'); }

let t = performance.now();
const user = await fetchUser(7);
const orders = await fetchOrders(7);
console.log('sequential:', Math.round(performance.now() - t), 'ms');

t = performance.now();
const [u2, o2] = await Promise.all([fetchUser(7), fetchOrders(7)]);
console.log('parallel:  ', Math.round(performance.now() - t), 'ms');

const results = await Promise.allSettled([fetchUser(7), fetchFlaky()]);
console.log(results.map(r => r.status));

try {
  await sleep(1000, null, { signal: AbortSignal.timeout(200) });
} catch (err) {
  console.log('aborted:', err.name);
}
```

```text
sequential: 255 ms
parallel:   150 ms
[ 'fulfilled', 'rejected' ]
aborted: AbortError
```

Takeaways:

- Two independent `await`s in a row take the **sum** of their times. Start both, then
  await them together with `Promise.all` to take the **max**.
- `Promise.all` rejects as soon as any input rejects. Use `Promise.allSettled` when a
  page can render with partial data.
- `Promise.race` settles with the first to settle; `Promise.any` with the first to
  *succeed*.
- `AbortSignal` is Node's standard cancellation mechanism. `fetch`, `timers/promises`,
  `fs.readFile`, `events.once`, streams, and child processes accept `{ signal }`.
  `AbortSignal.timeout(ms)` creates one that fires automatically.

### Limiting concurrency

`Promise.all(items.map(fetchSomething))` starts *everything* at once. With 10,000 items
you will overwhelm the remote service or exhaust sockets. A tiny worker-pool pattern:

```js
async function mapLimit(items, limit, fn) {
  const results = new Array(items.length);
  let next = 0;
  async function worker() {
    while (next < items.length) {
      const i = next++;           // safe: no other code runs between read and increment
      results[i] = await fn(items[i], i);
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}

const pages = await mapLimit(urls, 5, (url) => fetch(url).then(r => r.text()));
```

## How It Actually Works

A promise is an object with a state (`pending`, `fulfilled`, `rejected`) and a list of
reactions. Calling `.then(fn)` on a pending promise adds `fn` to that list; when the
promise settles, V8 queues each reaction as a **microtask**. Node drains the microtask
queue after every macrotask callback (timer, I/O callback, `setImmediate`), before the
event loop moves on.

`async/await` is built on this. When V8 hits `await p`, it suspends the function — saving
its local variables and position, much like a generator — and registers a reaction on
`p`. The function returns a pending promise to its caller right away. When `p` settles,
the microtask resumes the function where it left off. No thread is blocked; the "waiting"
is just a registered reaction.

Callback APIs in Node map directly onto libuv requests: `fs.readFile(path, cb)` submits
work and stores `cb`; libuv invokes it on completion. `fs/promises` does the same work
but resolves a promise instead of calling `cb`. `util.promisify(fn)` simply returns a
function that creates a promise and passes a callback resolving/rejecting it.

**Unhandled rejections.** If a promise rejects and no handler is attached by the time
the microtask queue drains, Node emits `unhandledRejection` on `process`. Since Node 15
the default is to treat it like an uncaught exception: print the error and **exit with
a non-zero code**. That is deliberate — a silently swallowed failure is worse than a
crash that a process manager restarts.

## Common mistakes

- **Forgetting `await`.** `const user = getUser(id)` gives you a pending promise;
  `user.name` is `undefined`. If the promise rejects, nothing catches it.
- **`array.forEach(async ...)`** doesn't wait for anything; `forEach` ignores returned
  promises. Use `for...of` with `await` (sequential) or `Promise.all(array.map(...))`
  (parallel).
- **Accidental serialization** — awaiting independent calls one after another.
- **Unbounded parallelism** with `Promise.all` over large arrays.
- **Mixing callbacks and promises** in one function, so errors go to two places (or
  none).
- **`catch` blocks that swallow errors** (`catch {}`) and hide real failures.

## Exercise

1. Write `readJsonFiles(paths)` that reads several JSON files in parallel and returns
   `{ ok: [...], failed: [{ path, error }] }` using `Promise.allSettled`.
2. Using `mapLimit`, fetch 20 pages from a public test API (or simulate with `sleep` and
   random delays) with a concurrency of 3. Log when each starts and finishes to confirm
   no more than 3 are ever in flight.
3. Wrap `dns.lookup` with `promisify` and add a 500 ms timeout using `Promise.race` and
   `timers/promises`. What happens to the underlying lookup when the timeout wins?
