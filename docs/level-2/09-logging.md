---
description: "Structured JSON logging in Node with Pino: levels, child loggers, pino-http request logs, request ids, redacting secrets, and why it stays fast."
---

# 09 · Structured Logging with Pino

`console.log('user ' + id + ' failed login')` is fine on your laptop. In production,
logs from dozens of processes flow into a system like Loki, Elasticsearch, CloudWatch,
or Datadog, and someone at 2 a.m. needs to answer "show me every error for request
`b7e1c2`" or "how many failed logins per minute?". That requires **structured logs**:
one JSON object per line, with consistent fields that machines can filter and aggregate.

**Pino** is the most widely used structured logger for Node, designed to add as little
overhead as possible to the event loop. (Winston is the other long-standing option; the
principles are the same.)

```bash
npm install pino pino-http
npm install -D pino-pretty     # human-readable output in development only
```

## Basics

```js title="logger-demo.js"
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: { service: 'orders-api' },
  redact: { paths: ['req.headers.authorization', '*.password', 'card.number'], censor: '[redacted]' },
});

logger.info('server starting');
logger.debug('this is hidden at level info');
logger.info({ orderId: 981, totalCents: 4599 }, 'order placed');

const reqLog = logger.child({ reqId: 'b7e1c2' });
reqLog.warn({ user: { email: 'ada@example.com', password: 'hunter2hunter2' } }, 'login failed');

try {
  JSON.parse('{');
} catch (err) {
  reqLog.error({ err }, 'could not parse payload');
}
```

Output (stack trimmed):

```text
{"level":30,"time":1790435212476,"service":"orders-api","msg":"server starting"}
{"level":30,"time":1790435212477,"service":"orders-api","orderId":981,"totalCents":4599,"msg":"order placed"}
{"level":40,"time":1790435212477,"service":"orders-api","reqId":"b7e1c2","user":{"email":"ada@example.com","password":"[redacted]"},"msg":"login failed"}
{"level":50,"time":1790435212477,"service":"orders-api","reqId":"b7e1c2","err":{"type":"SyntaxError","message":"Expected property name or '}' in JSON at position 1 (line 1 column 2)","stack":"SyntaxError: ..."},"msg":"could not parse payload"}
```

Things to notice:

- **Signature: `logger.level(mergingObject, message)`**. Data goes in the object as
  fields, not interpolated into the message. The message stays constant
  (`"order placed"`), so you can count occurrences; the data stays queryable
  (`orderId = 981`).
- **Levels are numbers**: trace 10, debug 20, info 30, warn 40, error 50, fatal 60.
  The `debug` line was dropped because the level was `info`.
- **Child loggers** carry context (`reqId`) into every line they write without repeating
  it.
- **Errors under the `err` key** are serialized with type, message, and stack (and
  `cause` chains).
- **Redaction** replaced the password. Redaction paths are compiled once at startup.

For development, pipe through pino-pretty: `node src/server.js | npx pino-pretty`.
Don't pretty-print in production; the log pipeline wants JSON.

## Request logging with pino-http

`pino-http` logs one line per completed request and attaches a child logger to each
request as `req.log`:

```js
import { pinoHttp } from 'pino-http';
app.use(pinoHttp({ logger }));
app.get('/tasks/:id', (req, res) => {
  req.log.info({ taskId: req.params.id }, 'loading task');
  res.json({ ok: true });
});
```

Running one request produced two lines — the handler's log and the completion log —
sharing the same `req.id`:

```text
{"level":30,"req":{"id":1,"method":"GET","url":"/tasks/7",...,"headers":{"authorization":"Bearer x",...}},"taskId":"7","msg":"loading task"}
{"level":30,"req":{"id":1,...},"res":{"statusCode":200,...},"responseTime":3,"msg":"request completed"}
```

Look closely: **the `Authorization` header was logged in full**. The default request
serializer includes all headers, so without a redact rule for
`req.headers.authorization` (and `req.headers.cookie`), every bearer token and session
cookie ends up in your log storage. Add redaction before your first deploy.

## Worked example: request ids across services

The Level 2 project generates or propagates a request id so that one user action can be
traced through logs of several services:

```js
app.use(pinoHttp({
  logger,
  genReqId: (req, res) => {
    const id = req.get('x-request-id') ?? randomUUID();
    res.setHeader('x-request-id', id);    // client can report it in a bug ticket
    return id;
  },
}));
```

When this service calls another, it forwards the header:
`fetch(url, { headers: { 'x-request-id': req.id } })`. Level 4 replaces this with
OpenTelemetry trace context, which does the same thing in a standardized way.

And in the error handler (lesson 05), `req.log.error({ err }, 'unhandled error')`
automatically includes the request id, method, and URL — the context you need to
reproduce the failure.

## What to log (and not)

Log:

- One line per request (method, route, status, duration) — pino-http does this.
- Significant business events: `order placed`, `payment failed`, `user registered`.
- Every 5xx with the full error.
- Startup configuration (**redacted**) and shutdown.

Don't log:

- Passwords, tokens, cookies, API keys, full card numbers, or other secrets.
- Personal data beyond what you need; logs are often retained longer and accessed more
  broadly than your database.
- Entire request/response bodies by default.
- Inside hot loops at `info` level.

## How It Actually Works

Pino is fast because it does as little as possible on the main thread:

1. **Level check first.** `logger.debug(...)` at level `info` is a no-op function: Pino
   replaces disabled level methods with `noop` when the level is set, so there is no
   serialization cost.
2. **Cheap serialization.** Pino builds the JSON line with string concatenation and a
   fast stringify path, and precomputes the static prefix (`"level"`, bindings from
   `base` and child loggers) once, rather than re-serializing it per call. A child
   logger's bindings are serialized when the child is created.
3. **Asynchronous-friendly output.** By default Pino writes to stdout via `sonic-boom`,
   a destination that buffers and uses `fs.write` on the file descriptor. Log shipping,
   formatting, or sending to a remote service should happen in **another process**
   (reading your stdout) or a **worker-thread transport** (`pino.transport(...)`), never
   inline in your request path.

This is also why the twelve-factor advice is to log to **stdout** and let the platform
(Docker, Kubernetes, systemd/journald) collect it: the app doesn't manage files,
rotation, or network sinks.

`redact` compiles the paths into a function at startup (via `fast-redact`), so redaction
costs a few property writes per log call, not a deep scan.

## Common mistakes

- **String-concatenated messages** with data baked in — unsearchable.
- **Logging secrets**, especially the default-logged `Authorization`/`Cookie` headers.
- **`console.log` in production code** alongside the logger: no levels, no structure.
- **Pretty-printing in production.**
- **Logging errors as strings** (`logger.error(err.message)`) — loses stack and cause.
  Use `{ err }`.
- **Debug logs at `info`** — noise that costs money in log storage.

## Exercise

1. Configure redaction for `req.headers.authorization`, `req.headers.cookie`, and
   `*.password` in the project's logger. Write a test that captures log output (pass a
   custom destination stream to `pino`) and asserts the token doesn't appear.
2. Set `customLogLevel` in pino-http so 4xx responses log at `warn` and 5xx at `error`.
3. After authentication, create `req.log = req.log.child({ userId: req.user.id })` and
   log a `task created` event from the handler. Check which log lines carry `userId` and
   which don't (inspect the completion line too), and explain the difference by reading
   how pino-http creates its per-request logger.
4. Pipe your server through `pino-pretty` in an `npm run dev` script, keeping `npm start`
   raw JSON.
