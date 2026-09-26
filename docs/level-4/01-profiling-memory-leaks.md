---
description: "Find why a Node service is slow or growing: CPU profiling with --cpu-prof and DevTools, reading flame graphs, heap snapshots, and hunting memory leaks."
---

# 01 · Performance Profiling & Memory Leaks

"The API got slow" and "the pods keep getting OOM-killed" are the two performance
complaints every Node team eventually receives. Guessing at causes wastes days. This
lesson is about **measuring**: CPU profiles to find where time goes, and heap measurements
and snapshots to find what memory is being kept alive.

The rule that saves the most time: **profile before optimizing**. The slow part is rarely
where you expect.

## CPU profiling

V8 has a sampling profiler built in. Every so often (roughly every millisecond by default)
it records the current JavaScript call stack. Functions that appear in many samples are
where the CPU time goes.

Ways to collect a profile:

- `node --cpu-prof app.js` — writes a `.cpuprofile` file when the process exits.
- `node --inspect app.js` then open `chrome://inspect` in Chrome, attach DevTools, and use
  the **Performance** (or Profiler) panel to record while you send load.
- Programmatically with the `node:inspector` module, e.g. from an admin-only endpoint.
- Third-party tools such as Clinic.js or `0x` wrap these and generate flame graphs.

## Worked example: profiling a slow report

```js title="slow.mjs"
// A "report" endpoint's core logic with two hidden inefficiencies
function buildReport(orders) {
  const byCustomer = {};
  for (const o of orders) {
    // 1. Re-sorting the whole array for every order: O(n^2 log n)
    const sorted = [...orders].sort((a, b) => a.amount - b.amount);
    const rank = sorted.indexOf(o);
    // 2. Building strings by repeated JSON round-trips
    const copy = JSON.parse(JSON.stringify(o));
    (byCustomer[copy.customer] ??= []).push({ ...copy, rank });
  }
  return byCustomer;
}

const orders = Array.from({ length: 3000 }, (_, i) => ({ id: i, customer: `c${i % 50}`, amount: (i * 7919) % 1000 }));
const t0 = performance.now();
buildReport(orders);
console.log(`report built in ${Math.round(performance.now() - t0)} ms`);
```

```text
$ node --cpu-prof --cpu-prof-dir=prof slow.mjs
report built in 910 ms
```

A `.cpuprofile` is JSON: a tree of call-frame nodes, plus the sequence of sampled node ids
and time deltas between samples. You can drag it into Chrome DevTools' Performance panel
for a flame chart, or summarize it with a few lines of code:

```js title="top.mjs"
// Summarize a .cpuprofile: self time per function, top 6
import { readFileSync, readdirSync } from 'node:fs';
const dir = process.argv[2];
const file = readdirSync(dir).find(f => f.endsWith('.cpuprofile'));
const { nodes, samples, timeDeltas } = JSON.parse(readFileSync(`${dir}/${file}`, 'utf8'));
const byId = new Map(nodes.map(n => [n.id, n]));
const self = new Map();
samples.forEach((id, i) => {
  const { functionName, url, lineNumber } = byId.get(id).callFrame;
  const key = `${functionName || '(anonymous)'} ${url.split('/').pop()}:${lineNumber + 1}`;
  self.set(key, (self.get(key) ?? 0) + (timeDeltas[i] ?? 0));
});
const total = [...self.values()].reduce((a, b) => a + b, 0);
for (const [k, us] of [...self].sort((a, b) => b[1] - a[1]).slice(0, 6)) {
  console.log(`${((us / total) * 100).toFixed(1).padStart(5)}%  ${k}`);
}
```

```text
$ node top.mjs prof
 77.2%  buildReport slow.mjs:2
 16.3%  (anonymous) slow.mjs:6
  2.4%  compileForInternalLoader realm:384
  1.4%  (program) :0
  0.8%  (garbage collector) :0
  0.2%  (idle) :0
```

The profile points straight at `buildReport` and the sort comparator on line 6 (built-ins
such as `Array.prototype.sort` and `indexOf` are attributed to their caller here). The
fix is algorithmic, not micro-optimization: sort **once**, look up ranks in a `Map`, and
replace the JSON round-trip with a shallow copy.

```js title="fast.mjs"
function buildReport(orders) {
  const rankOf = new Map(
    [...orders].sort((a, b) => a.amount - b.amount).map((o, i) => [o, i]),  // sort once
  );
  const byCustomer = {};
  for (const o of orders) {
    (byCustomer[o.customer] ??= []).push({ ...o, rank: rankOf.get(o) });   // shallow copy
  }
  return byCustomer;
}

const orders = Array.from({ length: 3000 }, (_, i) => ({ id: i, customer: `c${i % 50}`, amount: (i * 7919) % 1000 }));
const t0 = performance.now();
buildReport(orders);
console.log(`report built in ${(performance.now() - t0).toFixed(1)} ms`);
```

```text
report built in 1.6 ms
```

From 910 ms to under 2 ms on the same input. In a server, the first version would have
blocked the event loop for nearly a second per report request.

### Reading flame graphs

In a flame graph each box is a function; width is the share of samples in which it was on
the stack (inclusive time); children sit above (or below, depending on the tool) their
callers. Look for **wide plateaus**: a wide box with nothing on top is burning CPU itself
(self time). Common Node culprits: `JSON.parse`/`stringify` of large payloads,
synchronous crypto or compression, regexes, logging with pretty-printing, and O(n²) loops
hidden behind helper calls like `find` or `includes` inside a loop.

## Memory leaks

In a garbage-collected language, a "leak" means **objects stay reachable** that you no
longer need. The GC can only free unreachable objects. Typical Node leak sources:

- Module-level `Map`/object caches without eviction or TTL.
- Event listeners added per request to a long-lived emitter and never removed.
- Closures captured by timers (`setInterval`) that are never cleared.
- Arrays of "recent" items that are appended to but never trimmed.
- Promises that never settle, holding their closures.

### Worked example: a leak, observed

```js title="leak.mjs"
import { createServer } from 'node:http';
import { writeHeapSnapshot } from 'node:v8';

// "Cache" of per-request data that is never evicted: a classic leak
const seen = new Map();

const server = createServer((req, res) => {
  const id = `${Date.now()}-${Math.random()}`;
  seen.set(id, { headers: req.headers, url: req.url, body: Buffer.alloc(10_000).toString('hex') });
  res.end('ok');
});

server.listen(3500, async () => {
  const mb = (n) => (n / 1024 / 1024).toFixed(1);
  for (let round = 1; round <= 4; round++) {
    for (let batch = 0; batch < 20; batch++) {     // 20 batches of 100 concurrent requests
      await Promise.all(Array.from({ length: 100 }, () => fetch('http://localhost:3500/').then(r => r.text())));
    }
    globalThis.gc?.();
    const { heapUsed, rss } = process.memoryUsage();
    console.log(`after ${round * 2000} requests: heapUsed=${mb(heapUsed)} MB rss=${mb(rss)} MB entries=${seen.size}`);
  }
  console.log('snapshot written to', writeHeapSnapshot());
  server.close();
});
```

```text
$ node --expose-gc leak.mjs
after 2000 requests: heapUsed=50.0 MB rss=198.8 MB entries=2000
after 4000 requests: heapUsed=89.0 MB rss=254.1 MB entries=4000
after 6000 requests: heapUsed=127.7 MB rss=285.4 MB entries=6000
after 8000 requests: heapUsed=166.8 MB rss=325.1 MB entries=8000
snapshot written to Heap.20260926.204957.11778.0.001.heapsnapshot
```

Forcing a GC (`--expose-gc` and `gc()`, for diagnosis only) before measuring removes
garbage that simply hasn't been collected yet. `heapUsed` still climbs by ~39 MB per 2,000
requests — about 20 KB per request, which matches the 20,000-character hex string stored
for each one. That's a leak.

To find the culprit in a real app:

1. Take a heap snapshot after warm-up, apply load, take another (`v8.writeHeapSnapshot()`,
   the DevTools **Memory** tab, or `node --heapsnapshot-signal=SIGUSR2` to write one on a
   signal).
2. Load both in DevTools → Memory → select the second → **Comparison** view against the
   first. Sort by *# Delta* or *Size Delta*.
3. Pick a growing constructor (here: `(string)` and `Object`), open an instance, and read
   its **Retainers** panel: the chain of references keeping it alive. It will lead to
   `seen` → `Map` → the module scope.

For leaks that only show up in production, `--heapsnapshot-near-heap-limit=1` writes a
snapshot just before the process would run out of heap. Heap snapshots contain your data
(including secrets and personal data present in memory) — handle them like production
database dumps.

## How It Actually Works

**The sampling profiler** runs on a separate thread that periodically interrupts the main
thread and walks its stack (JavaScript frames, plus markers for native code, GC, and
idle). Sampling is cheap enough to use briefly in production, but it's statistical:
functions that run for less than the sampling interval may not show up individually, and
inlined functions are attributed to their caller — which is why `indexOf` and `sort` didn't
appear by name above.

**V8's heap** is generational. New objects are allocated in the **young generation** (a
small space collected very frequently by a fast copying "scavenger"). Objects that survive
a couple of scavenges are promoted to the **old generation**, collected by a
mark-sweep-compact collector that runs incrementally and concurrently where possible.
Marking starts from **roots** (globals, the stack, handles held by native code) and
follows references; everything not marked is garbage. A leak is therefore always a path
from a root to your objects — exactly what the Retainers view shows.

The old generation has a size limit (derived from system memory and, in containers, the
cgroup memory limit on recent Node versions); `--max-old-space-size=<MB>` overrides it.
Near the limit, GC runs more and more often (the process gets slower before it dies), and
finally V8 aborts with "JavaScript heap out of memory". Buffers are allocated outside the
V8 heap, so a Buffer leak shows up in `rss`/`arrayBuffers` rather than `heapUsed`.

## Common mistakes

- **Optimizing without a profile** — rewriting code that accounts for 2% of time.
- **Profiling a cold process** — the first seconds include module loading and JIT
  warm-up. Profile under realistic, sustained load.
- **Reading `rss` alone** — it includes code, stacks, and memory the allocator hasn't
  returned to the OS. Watch `heapUsed` after GC, and `external`/`arrayBuffers` for buffers.
- **Raising `--max-old-space-size`** to "fix" a leak. It only delays the crash.
- **Unbounded in-memory caches.** Use an LRU with a size limit (e.g. `lru-cache`) or Redis.
- **Leaving `--inspect` open on a public interface** — the inspector allows arbitrary code
  execution. Bind to localhost and tunnel if needed.

## Exercise

1. Profile the Level 2 tasks API under load (e.g. `npx autocannon -c 50 -d 20
   http://localhost:3000/tasks` with a token header) and identify its top three functions
   by self time.
2. Fix `leak.mjs` with a bounded LRU cache (max 500 entries) and rerun it to show
   `heapUsed` plateauing.
3. Create a leak by adding a `process.on('SIGTERM', ...)` listener inside a request
   handler. Find it with two heap snapshots, and note the `MaxListenersExceededWarning`.
4. Run `node --max-old-space-size=64 leak.mjs` and observe how the process fails. Then
   add `--heapsnapshot-near-heap-limit=1` and inspect the generated snapshot.
