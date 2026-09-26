---
description: "The Node.js event loop in depth: libuv phases, process.nextTick vs promise microtasks, setImmediate vs setTimeout, starvation, and measuring loop delay."
---

# 01 · The Event Loop in Depth

Level 1 introduced the event loop as "the thing that runs callbacks when I/O finishes".
That is enough to build APIs, but not enough to explain why a timer fired late, why a
`setImmediate` beat a `setTimeout(fn, 0)` on one run and lost on the next, or why a
recursive `process.nextTick` froze your server. This lesson opens the loop up.

## The phases

libuv's loop repeats these phases in order. Each phase has a queue of callbacks; the loop
runs the callbacks that are ready, then moves on.

```text
   ┌───────────────────────────┐
┌─>│          timers           │  setTimeout / setInterval callbacks whose time has come
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  some system callbacks deferred from the previous iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  internal
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  wait for I/O; run socket, fs-completion, etc. callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  'close' events, e.g. socket.on('close')
   └───────────────────────────┘
```

Two extra queues are **not** phases. They belong to Node (and V8), and they are drained
**after every single callback**, in every phase:

1. the **`process.nextTick` queue**, then
2. the **microtask queue** (promise reactions, `queueMicrotask`, `await` continuations).

If a nextTick callback schedules another nextTick, or a microtask queues another
microtask, those run too before the loop proceeds. The loop only advances when both
queues are empty.

## Worked example: predicting the order

```js title="order.mjs"
import { readFile } from 'node:fs';

console.log('sync start');

setTimeout(() => console.log('timeout 0'), 0);
setImmediate(() => console.log('immediate'));

Promise.resolve().then(() => console.log('promise.then'));
process.nextTick(() => console.log('nextTick'));
queueMicrotask(() => console.log('queueMicrotask'));

readFile(import.meta.filename, () => {
  console.log('fs callback (poll phase)');
  setTimeout(() => console.log('  timeout inside I/O'), 0);
  setImmediate(() => console.log('  immediate inside I/O'));
  process.nextTick(() => console.log('  nextTick inside I/O'));
  Promise.resolve().then(() => console.log('  promise inside I/O'));
});

console.log('sync end');
```

Output of `node order.mjs` on a test run:

```text
sync start
sync end
promise.then
queueMicrotask
nextTick
timeout 0
immediate
fs callback (poll phase)
  nextTick inside I/O
  promise inside I/O
  immediate inside I/O
  timeout inside I/O
```

The same code saved as a CommonJS file (`order.cjs`, with `require` and `__filename`)
printed this instead for the first five lines:

```text
sync start
sync end
nextTick
promise.then
queueMicrotask
```

Walk through it:

- **Synchronous code runs to completion first.** Nothing else can interleave.
- **nextTick vs promises at the top level depends on the module system.** In CommonJS,
  the main script runs as a plain function call; when it returns, Node drains nextTicks,
  then microtasks. An ES module, however, is *evaluated inside a promise job* — so V8 keeps
  draining the microtask queue it is already in (`promise.then`, `queueMicrotask`) before
  control returns to Node, which then processes nextTicks. Inside callbacks (the I/O
  callback above), both module types behave the same: **nextTick first, then promises**.
- **`timeout 0` vs `immediate` in the main module is not guaranteed.** On another run of
  this very script the two swapped places. `setTimeout(fn, 0)` is really 1 ms; whether
  that millisecond has elapsed by the time the loop first checks timers depends on how
  long startup took.
- **Inside an I/O callback, `setImmediate` always wins.** We are in the poll phase; the
  next phase is *check* (immediates), and timers only come around on the next iteration.
  If you need "run after this I/O, before anything else", `setImmediate` is the tool.

## Starvation

Because the nextTick and microtask queues are drained completely before the loop
continues, you can starve the loop without any obvious blocking loop:

```js title="starve.mjs"
let ticks = 0;
const start = Date.now();
setTimeout(() => console.log(`timer ran after ${Date.now() - start} ms, ${ticks} nextTicks later`), 0);
function spin() {
  if (Date.now() - start < 300) { ticks++; process.nextTick(spin); }
}
spin();
```

```text
timer ran after 300 ms, 3815916 nextTicks later
```

Each `spin` is quick, but they never let the loop reach the timers phase. The same
happens with a promise chain that keeps resolving synchronously-available values in a
loop. `setImmediate` does not have this problem: immediates queued *during* the check
phase run on the next iteration, so I/O gets a turn in between.

## Measuring event-loop delay

The most useful health metric for a Node service is **event-loop delay**: how late
callbacks run compared to when they should. `perf_hooks` measures it with a histogram:

```js title="lag.mjs"
import { monitorEventLoopDelay } from 'node:perf_hooks';

const h = monitorEventLoopDelay({ resolution: 10 });
h.enable();

// Simulate a request handler that does 200 ms of synchronous work every second
const timer = setInterval(() => {
  const end = Date.now() + 200;
  while (Date.now() < end) {}
}, 1000);

setTimeout(() => {
  h.disable();
  clearInterval(timer);
  const ms = (ns) => (ns / 1e6).toFixed(1);
  console.log(`event loop delay  p50=${ms(h.percentile(50))}ms  p99=${ms(h.percentile(99))}ms  max=${ms(h.max)}ms`);
}, 3500);
```

```text
event loop delay  p50=12.1ms  p99=202.6ms  max=204.7ms
```

The median looks fine; the p99 reveals that some callbacks waited ~200 ms. In a server,
that is 200 ms added to *every* request that arrived during the blocking work. Export
this histogram as a metric (Level 4 observability) and alert on its p99.

## How It Actually Works

`uv_run()` is a loop in C. Simplified:

```c
while (loop_alive(loop)) {
  uv__update_time(loop);        // cache "now" once per iteration
  uv__run_timers(loop);         // timers phase: min-heap ordered by expiry
  uv__run_pending(loop);
  uv__run_idle(loop);
  uv__run_prepare(loop);
  timeout = compute_poll_timeout(loop);   // 0 if immediates pending, else until next timer
  uv__io_poll(loop, timeout);   // epoll_wait / kevent / GetQueuedCompletionStatus
  uv__run_check(loop);          // setImmediate
  uv__run_closing_handles(loop);
}
```

- **Timers** are kept in a min-heap keyed by expiry time; the phase pops every timer that
  has expired. A timer's delay is a *minimum*, never a guarantee.
- **The poll phase is where Node sleeps.** It calls the OS readiness API with a timeout:
  zero if immediates are waiting, otherwise until the nearest timer, or indefinitely if
  there are no timers. The kernel wakes the thread when a socket has data or when a
  thread-pool job signals completion (thread-pool results are delivered through an
  internal async handle, which is just another fd in the poll set).
- **`loop_alive`** is true while there are referenced handles (servers, sockets, timers)
  or pending requests. `unref()` excludes a handle from this count.

Node's C++ layer wraps every callback it invokes from libuv in a helper
(`InternalCallbackScope`) that, when the callback returns, runs the nextTick queue and
then asks V8 to run its microtask checkpoint. That is the precise mechanism behind
"drained after every callback". `process.nextTick` is purely a Node concept, implemented
in JavaScript as an array-backed queue; microtasks are V8's.

## Common mistakes

- **Using `process.nextTick` for "later"** in application code. Prefer `setImmediate` or
  `queueMicrotask`; reserve nextTick for library code that must emit events after the
  caller attached listeners.
- **Expecting precise timers** — `setTimeout(fn, 100)` runs *at least* 100 ms later.
- **Relying on `setTimeout(0)` vs `setImmediate` order** in the main module.
- **Monitoring only average latency** — loop blocking shows up in tail latency (p99).
- **Chunking CPU work with microtasks** (`await null` in a loop) — it doesn't let I/O
  run. Yield with `await setImmediate()` from `node:timers/promises`, or move the work to
  a worker thread (lesson 04).

## Exercise

1. Before running it, write down the output order you expect for a script that nests a
   `setImmediate` inside a `setTimeout` inside a `process.nextTick`. Run it; explain every
   line.
2. Convert `starve.mjs` to use `setImmediate` instead of `process.nextTick` and measure
   how late the timer runs.
3. Write a function that sums the numbers 1..500,000,000 in chunks of 10 million,
   yielding with `await setImmediate()` between chunks. Run an HTTP server alongside and
   compare `/health` latency with and without chunking.
4. Add `monitorEventLoopDelay` to the Level 2 API and expose p50/p99 at `GET /metrics/loop`.
