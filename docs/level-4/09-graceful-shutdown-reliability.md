---
description: "Keep Node services reliable through deploys and failures: graceful SIGTERM shutdown, draining connections, readiness, timeouts, and crash-only design."
---

# 09 · Graceful Shutdown & Reliability

Every deploy kills your processes. So do autoscaling, node maintenance, and crashes. A
service that drops in-flight requests every time it's replaced will show error spikes on
every release, and teams start fearing deploys. Reliability at the process level comes down
to a few habits: **shut down gracefully**, **bound every wait with a timeout**, **fail fast
on corrupted state**, and **assume you can be killed at any moment anyway**.

## The shutdown sequence

When a platform stops an instance, it sends `SIGTERM`, waits a grace period, then sends
`SIGKILL` (which can't be caught). Within the grace period a good Node service:

1. **Stops being "ready"** so the load balancer stops sending new requests.
2. **Stops accepting new connections** (`server.close()`).
3. **Finishes in-flight requests** (and in workers: the current job).
4. **Closes idle keep-alive connections**, so clients reconnect to a healthy instance.
5. **Closes resources in order** — DB pools, Redis, message consumers, telemetry exporters
   (to flush buffered spans/logs).
6. **Exits**, or is force-exited by its own deadline shortly before `SIGKILL` would arrive.

## Worked example: draining an in-flight request

```js title="server.mjs"
import { createServer } from 'node:http';
import { setTimeout as sleep } from 'node:timers/promises';

const log = (msg) => console.log(`${new Date().toISOString().slice(17, 23)} ${msg}`);
let shuttingDown = false;

const server = createServer(async (req, res) => {
  if (req.url === '/readyz') {
    res.writeHead(shuttingDown ? 503 : 200).end(shuttingDown ? 'draining' : 'ready');
    return;
  }
  if (req.url === '/slow') {
    log('slow request started');
    await sleep(1500);                        // e.g. a report being generated
    res.end('slow result\n');
    log('slow request finished');
    return;
  }
  res.end('ok\n');
});

// Pretend resources that must be closed in order
const db = { async end() { await sleep(100); log('db pool closed'); } };

async function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  log(`${signal} received: draining`);

  // Hard deadline: never hang forever (keep below the platform's kill timeout)
  const force = setTimeout(() => { log('forced exit'); process.exit(1); }, 8000);
  force.unref();

  // Optional: keep accepting for a moment so the load balancer notices /readyz = 503
  // await sleep(Number(process.env.DRAIN_DELAY_MS ?? 0));

  server.close(async (err) => {              // stop accepting; callback when all connections end
    log('http server closed');
    await db.end();
    log('clean exit');
    process.exitCode = err ? 1 : 0;
  });
  server.closeIdleConnections();              // drop idle keep-alive sockets now
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
server.listen(3800, () => log(`listening, pid ${process.pid}`));
```

The test: start a 1.5-second request, send `SIGTERM` 0.3 seconds into it, then try a new
request during the drain.

```bash
node server.mjs & SP=$!
curl -s localhost:3800/slow > slow.out &
sleep 0.3; kill -TERM $SP
sleep 0.2; curl -s -o /dev/null -w "new request during drain: %{http_code} (curl exit %{exitcode})\n" localhost:3800/
```

```text
new request during drain: 000 (curl exit 7)
10.241 listening, pid 12278
10.669 slow request started
10.960 SIGTERM received: draining
12.178 slow request finished
12.181 http server closed
12.284 db pool closed
12.284 clean exit
client got: slow result
```

- The in-flight request **completed normally** after the signal; its client got its result.
- A new connection during the drain was refused (curl exit code 7: couldn't connect),
  because the listening socket was closed. Behind a load balancer this is where step 1
  matters: if the balancer still routes to this instance for a few seconds after SIGTERM,
  those requests fail. That's why many setups first flip `/readyz` to 503 and wait a short
  drain delay (the commented `DRAIN_DELAY_MS` line) before calling `server.close()`.
  Kubernetes users often add a `preStop` sleep for the same reason.
- The server closed only after the last connection ended, then the DB pool closed, then the
  process exited with code 0 — by running out of work, not by `process.exit()`.
- The force-exit timer is `unref()`ed so it doesn't itself keep the process alive.

In Express, `app.listen()` returns the same `http.Server`, so the pattern is identical.
With WebSockets, send close frames (code 1001, "going away") so clients reconnect
elsewhere. With queue workers, stop claiming new jobs and let the current one finish.

## Timeouts everywhere

Most outages in Node services aren't crashes; they're **hangs**: requests waiting on
something that will never answer, piling up until memory or sockets run out. Put a bound on
every wait:

| Wait | Bound |
|------|-------|
| Outbound HTTP | `AbortSignal.timeout(ms)` on every `fetch` (lesson 06) |
| Database | pool acquire timeout, `statement_timeout` in Postgres |
| Redis | client command timeout |
| Incoming requests | `server.requestTimeout`, `server.headersTimeout` (L1-08) |
| Whole request | a per-request deadline middleware; pass the signal down |
| Shutdown | the force-exit timer above |

Keep them consistent: a request deadline of 5 s is meaningless if a DB query inside it can
take 30 s.

## Reliability patterns

- **Crash-only thinking.** Design so that being killed at any instant is safe: use
  transactions, idempotent jobs, and an outbox for events. Then graceful shutdown is an
  optimization, not the only thing protecting your data.
- **Fail fast on programmer errors.** Log and exit on `uncaughtException` (Level 2,
  lesson 05) — combine it with the shutdown routine so in-flight requests on other
  connections still get a chance to finish, but with a short deadline.
- **Load shedding.** When event-loop delay is very high, it can be better to return 503
  quickly for some requests than to serve all of them slowly.
- **Bulkheads.** Separate pools/queues for different dependencies or priorities so that one
  slow dependency can't consume every socket or worker.
- **Idempotency** at every boundary that might be retried.

## How It Actually Works

**Signals.** `SIGTERM` and `SIGINT` are delivered by the OS to the process. libuv installs
a real signal handler that only writes to an internal pipe; the event loop then sees the
pipe become readable and runs your JavaScript `process.on('SIGTERM')` handler on the main
thread. That means a handler can't run while JavaScript is busy — a process stuck in a
tight synchronous loop can't shut down gracefully, and will be `SIGKILL`ed. Registering a
handler replaces the default behavior (terminating), which is why your handler must
eventually lead to an exit.

**`server.close()`** closes the *listening* socket immediately (no new connections) and
invokes its callback once all existing connections have closed. Keep-alive connections
count: an idle keep-alive socket from a client would hold the server open until the
keep-alive timeout expires. `server.closeIdleConnections()` closes sockets that are not
currently processing a request; in current Node versions `close()` also closes idle
connections, and responses sent during shutdown include `Connection: close` so clients
don't reuse the socket. `server.closeAllConnections()` is the blunt instrument for the
force-exit path.

**Why exit naturally instead of `process.exit(0)` right after cleanup?** `process.exit()`
terminates immediately, even if stdout (for example, a pipe to a log collector) still has
buffered data. Letting the event loop empty guarantees pending writes flush. Setting
`process.exitCode` chooses the code without cutting anything short.

## Common mistakes

- **No SIGTERM handler**, especially combined with running as PID 1 in a container (the
  signal may be ignored; see lesson 07).
- **`CMD npm start`** so the signal never reaches Node.
- **Calling `process.exit()` immediately on SIGTERM**, dropping in-flight requests.
- **No force-exit deadline**, so a stuck connection keeps the instance alive until SIGKILL.
- **Closing the DB pool before the HTTP server drains**, failing the in-flight requests
  you were trying to finish.
- **Forgetting to flush telemetry** on shutdown.
- **Missing timeouts** on outbound calls and queries.

## Exercise

1. Add the shutdown sequence to the Level 2 tasks API: readiness flips to 503, a
   configurable drain delay, `server.close()`, `db.destroy()`, and a force-exit timer.
2. Write an automated test that spawns the server as a child process, starts a slow request,
   sends `SIGTERM`, and asserts the request succeeds and the child exits with code 0.
3. Add a request-deadline middleware that creates an `AbortController` per request, aborts
   after 5 seconds, and passes `req.signal` to downstream `fetch` calls.
4. Simulate a stuck process with a `while (true) {}` handler and observe that SIGTERM
   can't be handled. Explain what the platform does next.
