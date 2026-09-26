---
description: "Use every CPU core with node:cluster or multiple processes: how connections are distributed, restarting workers, shared state, and when to skip it."
---

# 05 · Clustering & Using Every Core

One Node process runs your JavaScript on one core. On an 8-core server, a single process
leaves most of the CPU idle once request handling (JSON serialization, validation,
templating) becomes the bottleneck. The fix is to run **several copies of the process**
and spread connections across them. There are two ways:

1. **`node:cluster`** — a primary process forks workers that share one listening port.
2. **Separate processes behind a load balancer** — the approach containers and
   Kubernetes use: run N replicas, each a plain single-process Node app.

## node:cluster in 30 lines

```js title="cluster.mjs"
import cluster from 'node:cluster';
import { createServer } from 'node:http';
import { availableParallelism } from 'node:os';

const WORKERS = Number(process.env.WORKERS ?? availableParallelism());

if (cluster.isPrimary) {
  console.log(`primary ${process.pid} starting ${WORKERS} workers`);
  for (let i = 0; i < WORKERS; i++) cluster.fork();

  cluster.on('exit', (worker, code, signal) => {
    if (worker.exitedAfterDisconnect) return;          // intentional shutdown
    console.log(`worker ${worker.process.pid} died (${signal ?? code}); starting a replacement`);
    cluster.fork();
  });

  process.on('SIGTERM', () => {
    for (const w of Object.values(cluster.workers)) w.disconnect();  // finish in-flight, then exit
  });
} else {
  createServer((req, res) => {
    if (req.url === '/crash') process.exit(1);          // simulate a fatal bug
    res.end(`handled by worker ${process.pid}\n`);
  }).listen(3200);
}
```

Running it with three workers and calling it a few times:

```text
$ WORKERS=3 node cluster.mjs
primary 11406 starting 3 workers

$ for i in 1 2 3 4 5 6; do curl -s localhost:3200/; done
handled by worker 11408
handled by worker 11409
handled by worker 11410
handled by worker 11408
handled by worker 11409
handled by worker 11410

$ curl -s localhost:3200/crash       # (primary logs:)
worker 11408 died (1); starting a replacement
$ curl -s localhost:3200/
handled by worker 11409
```

Requests rotate round-robin across workers, and when one dies the primary replaces it —
the port stays open the whole time because the primary owns it.

## What clustering does not give you

**Shared memory.** Each worker is a separate process with its own heap. An in-memory
cache, rate-limit counter, session store (`MemoryStore`), or WebSocket connection list
exists **per worker**. With three workers, a user may log in on worker A and hit worker B
next, which has never heard of them. Anything that must be shared goes to an external
store — Redis (next lesson) or the database.

**Sticky connections.** Round-robin is per *connection*, not per user. Protocols that need
a client to keep reaching the same process (some long-polling fallbacks in real-time
libraries) require "sticky sessions" configured at the load balancer or via a helper
library.

## cluster vs containers

| | `node:cluster` | N single-process replicas |
|--|--|--|
| Where | one machine/VM | one or many machines |
| Restart on crash | primary forks a new worker | orchestrator (Kubernetes, ECS, systemd) restarts it |
| Load balancing | inside Node | external load balancer / service mesh |
| Rolling deploy | you implement it | the platform does it |
| Config complexity | code in your app | infrastructure config |

If you deploy to Kubernetes or a similar platform, the usual advice is to **not** use
`cluster`: run one Node process per container, give each container about one CPU, and
scale the replica count. The platform already provides restarts, health checks, and load
balancing, and one-process containers are simpler to reason about (memory limits,
signals, logs). `cluster` (or a process manager like PM2 that uses it) remains useful on
plain VMs.

Either way, remember the per-process resources you multiply: **database pool size ×
process count** must fit the database's connection limit.

## How It Actually Works

`cluster.fork()` spawns a new Node process running the same script, with an IPC channel
to the primary and an environment variable marking it as a worker (so `cluster.isPrimary`
is false there).

When a worker calls `server.listen(3200)`, the call doesn't bind a socket directly.
Instead the worker asks the primary over IPC. Then one of two scheduling policies applies:

- **Round-robin (`SCHED_RR`, default on all platforms except Windows):** the primary
  creates the listening socket and `accept()`s every connection itself, then passes the
  connected socket's **file descriptor** to a worker over the IPC channel (Unix domain
  sockets can transfer file descriptors between processes). The worker wraps it as a
  normal `net.Socket`. That's the rotation you saw in the output.
- **Shared handle (`SCHED_NONE`):** the primary creates the listening socket and sends
  *that* handle to each worker; all workers call `accept()` on it, and the kernel decides
  who wins. This can be very uneven because the OS tends to wake the same busy process.

Set the policy with `cluster.schedulingPolicy` or the `NODE_CLUSTER_SCHED_POLICY`
environment variable (`rr` or `none`).

`worker.disconnect()` closes the worker's servers (stop accepting new connections),
waits for existing connections to end, then closes the IPC channel so the worker can
exit. `exitedAfterDisconnect` lets the `exit` handler distinguish that from a crash.

An alternative without cluster is the `SO_REUSEPORT` socket option (on Linux), which lets
several independent processes bind the same port and has the kernel load-balance between
them; Node exposes it as `server.listen({ port, reusePort: true })` on supporting
platforms and versions. (On the macOS machine used for this lesson, that call failed with
`ENOTSUP`; treat it as a Linux-oriented option.)

## Common mistakes

- **In-process state** (sessions, caches, counters, socket lists) assumed to be global.
- **Forking more workers than cores** — context switching makes it slower, not faster.
  Also leave headroom for the libuv pool and GC threads.
- **Clustering inside containers** that are also horizontally scaled, doubling the layers
  of process management.
- **Crash loops** — a worker that dies on startup gets re-forked forever. Add backoff or a
  restart limit.
- **Running scheduled jobs in every worker** — a cron-like timer in cluster workers fires
  N times. Run it in one place (primary, or a separate process).

## Exercise

1. Run `cluster.mjs` with as many workers as `os.availableParallelism()`. Use a load tool
   (`npx autocannon localhost:3200`) against a handler that does ~5 ms of CPU work, and
   compare requests/second with `WORKERS=1`.
2. Add an in-memory hit counter and a `/count` route. Show that the counts diverge across
   workers, then fix it by asking the primary for a total via IPC (`process.send` /
   `worker.on('message')`).
3. Implement a zero-downtime restart: on `SIGUSR2`, the primary replaces workers one at a
   time, forking a new one and waiting for its `'listening'` event before disconnecting an
   old one.
4. Add exponential backoff to the restart logic if a worker dies within 5 seconds of
   starting.
