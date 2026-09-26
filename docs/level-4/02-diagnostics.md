---
description: "Diagnose Node services in production: diagnostic reports, AsyncLocalStorage context, diagnostics_channel, trace flags, and the inspector used safely."
---

# 02 · Diagnostics in Production

Profilers (previous lesson) answer "where does time go?" under controlled conditions.
Production problems are messier: a process that crashed at 3 a.m., requests that hang
without an error, a warning that appears once a day, or a log line you can't tie to the
request that caused it. Node ships a set of diagnostic tools built for exactly these
situations, and most of them cost nothing until you turn them on.

## Diagnostic reports: a snapshot of the whole process

A **diagnostic report** is a JSON document describing the process at a moment: Node and
OS versions, the JavaScript and native stacks, heap statistics, resource usage, every
libuv handle (sockets, timers, servers), environment variables, and loaded shared
libraries.

```js title="report.mjs"
const report = process.report.getReport();
console.log('node', report.header.nodejsVersion, '| cpus', report.header.cpus.length);
console.log('heap limit MB', Math.round(report.javascriptHeap.memoryLimit / 1024 / 1024));
console.log('active handles:', report.libuv.filter(h => h.is_active).map(h => h.type).join(', '));
console.log('top of JS stack:', report.javascriptStack.stack[0]);
```

```text
node v26.3.0 | cpus 8
heap limit MB 2144
active handles: async, async, check, prepare, check, async
top of JS stack: at Object.getReport (node:internal/process/report:39:13)
```

The most valuable use is automatic reports when things go wrong:

```bash
mkdir -p /var/reports
node --report-uncaught-exception --report-on-fatalerror \
     --report-on-signal --report-directory=/var/reports app.js
```

With a script that dereferences `null` inside a timer, the process printed
`Writing Node.js report to file: report.<date>.<pid>...json` before exiting, and the file
recorded `event: Exception` with the message
`TypeError: Cannot read properties of null (reading 'x')` plus the full stack.
(The directory must already exist; with a missing directory, Node printed
`Failed to open Node.js report file` and wrote nothing.)

`--report-on-signal` (default signal `SIGUSR2`) lets you ask a **hung** process for a
report without killing it: `kill -USR2 <pid>`. The `libuv` section then shows which
sockets and timers are open — often enough to see "4,000 open sockets to the payments
service" and understand the hang.

Reports contain environment variables — i.e. potentially secrets. Newer versions
support excluding them (`--report-exclude-env`); check your version's docs and treat
reports as sensitive files either way.

## AsyncLocalStorage: context that follows the request

Logs from deep inside your code ("loaded user 1") are useless if you can't tell which
request produced them. Passing a `reqId` through every function signature is tedious.
`AsyncLocalStorage` stores a value that automatically **follows the asynchronous flow** —
through `await`, timers, and callbacks — so any function can read the context of the
request it is working for:

```js title="als.mjs"
import { AsyncLocalStorage } from 'node:async_hooks';
import { createServer } from 'node:http';
import { randomUUID } from 'node:crypto';
import { setTimeout as sleep } from 'node:timers/promises';

const context = new AsyncLocalStorage();

// Any function, however deep, can read the current request's context without it being passed in
function log(msg) {
  const store = context.getStore();
  console.log(JSON.stringify({ reqId: store?.reqId ?? null, msg }));
}

async function loadUser(id) {
  await sleep(Math.random() * 20);        // pretend DB call; other requests interleave here
  log(`loaded user ${id}`);
}

const server = createServer((req, res) => {
  context.run({ reqId: randomUUID().slice(0, 8) }, async () => {
    log(`start ${req.url}`);
    await loadUser(req.url.slice(1));
    log('done');
    res.end('ok');
  });
});

server.listen(3600, async () => {
  await Promise.all(['/1', '/2'].map(p => fetch(`http://localhost:3600${p}`).then(r => r.text())));
  log('outside any request');
  server.close();
});
```

```text
{"reqId":"b95e89c4","msg":"start /1"}
{"reqId":"0eea4e30","msg":"start /2"}
{"reqId":"b95e89c4","msg":"loaded user 1"}
{"reqId":"b95e89c4","msg":"done"}
{"reqId":"0eea4e30","msg":"loaded user 2"}
{"reqId":"0eea4e30","msg":"done"}
{"reqId":null,"msg":"outside any request"}
```

The two requests interleaved (both started before either finished), yet every log line
carries the right id. This is the mechanism under many logging and tracing libraries —
OpenTelemetry's context propagation in Node is built on it (lesson 08). A common pattern
is to create a Pino child logger per request and store it in AsyncLocalStorage.

## diagnostics_channel: observe without modifying code

`node:diagnostics_channel` is a publish/subscribe system for diagnostic events. Node
core publishes events for HTTP clients and servers, `net`, `undici` (fetch), and more;
libraries can publish their own. Tools subscribe without monkey-patching:

```js title="dc.mjs"
import diagnostics_channel from 'node:diagnostics_channel';
import { createServer } from 'node:http';

// Subscribe to Node's built-in HTTP server channel without touching the app's code
diagnostics_channel.subscribe('http.server.response.finish', ({ request, response }) => {
  console.log(`[observer] ${request.method} ${request.url} -> ${response.statusCode}`);
});

const server = createServer((req, res) => { res.statusCode = req.url === '/missing' ? 404 : 200; res.end(); });
server.listen(3601, async () => {
  for (const p of ['/a', '/missing']) await fetch(`http://localhost:3601${p}`);
  server.close();
});
```

```text
[observer] GET /a -> 200
[observer] GET /missing -> 404
```

When nobody subscribes, publishing is close to free, which is why it's safe for libraries
to instrument hot paths this way.

## Quick diagnostic flags

| Flag / variable | What it tells you |
|-----------------|-------------------|
| `--trace-warnings` | stack trace for every process warning (e.g. `MaxListenersExceededWarning`) |
| `--trace-uncaught` | where an uncaught exception was *thrown*, even for non-`Error` values |
| `--trace-deprecation` | stack traces for deprecation warnings |
| `--unhandled-rejections=strict` | make unhandled rejections fail fast everywhere |
| `NODE_DEBUG=http,net` | verbose internal debug logs from core modules |
| `--trace-gc` | a line per garbage collection with timing |
| `--heapsnapshot-signal=SIGUSR2` | write a heap snapshot on a signal |
| `--inspect=127.0.0.1:9229` | attach Chrome DevTools/VS Code |

Attaching the inspector to a production process is sometimes the fastest route to an
answer, but the inspector port allows **arbitrary code execution**. Never bind it to a
public interface; use `127.0.0.1` and an SSH tunnel or `kubectl port-forward`. You can
also activate it at runtime with `kill -USR1 <pid>` on Unix, which makes a running
process start listening on 127.0.0.1:9229.

## A triage checklist

1. **Crash?** Read the last log lines and the diagnostic report (stack, heap stats).
2. **Slow?** Check event-loop delay (L3-01). High delay → CPU-bound code → CPU profile.
   Low delay but slow responses → waiting on something → look at downstream latency,
   pool saturation (DB pool waiting count), thread-pool saturation.
3. **Memory growing?** `heapUsed` after GC rising → heap snapshots (lesson 01). `rss`
   rising but heap flat → buffers or native memory.
4. **Hung?** `kill -USR2` for a report; inspect open handles and pending requests.

## How It Actually Works

**AsyncLocalStorage** relies on V8's *continuation-preserved embedder data* and Node's
async context tracking: when an asynchronous operation is created (a promise reaction,
a timer, an I/O request), Node records the current context with it; when its callback
later runs, Node restores that context before calling your code. `context.run(store, fn)`
sets the context for everything `fn` synchronously starts, and therefore for everything
those operations later start in turn. Older Node versions implemented this with
`async_hooks` callbacks on every resource creation, which had measurable overhead; newer
versions take the cheaper V8-integrated path.

**Diagnostic reports** are generated in C++ inside the process. On a fatal error (such as
heap out of memory) JavaScript can't run anymore, which is why the report writer is
native: it walks V8's stack and heap statistics and libuv's handle list directly.

**diagnostics_channel** channels are named objects holding a list of subscriber
functions. `channel.publish(msg)` checks `hasSubscribers` first and returns immediately
if there are none; otherwise it calls subscribers synchronously.

## Common mistakes

- **Enabling nothing until after an incident.** Turn on report-on-fatal-error and
  uncaught-exception reports now; they cost nothing until triggered.
- **Losing request context** in logs, making concurrent requests indistinguishable.
- **Exposing the inspector** on `0.0.0.0`.
- **Shipping reports and snapshots** to places with broader access than production data.
- **Heavy work in diagnostics_channel subscribers** — they run synchronously in the hot
  path.

## Exercise

1. Add `--report-uncaught-exception --report-on-signal --report-directory` to the Level 2
   project's start script. Trigger a report with `kill -USR2` and find the listening socket
   and the DB pool's sockets in the `libuv` section.
2. Refactor the Level 2 logger so handlers call `getLogger()` (backed by AsyncLocalStorage)
   instead of `req.log`, and prove concurrent requests keep separate ids.
3. Subscribe to `http.client.request.start` and log every outgoing HTTP call your service
   makes with its host and path.
4. Write a script that leaks a `setInterval` per request and use a diagnostic report to
   count the timers after 100 requests.
