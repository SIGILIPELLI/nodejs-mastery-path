---
description: "Handle errors properly in Node and Express 5: operational vs programmer errors, HttpError classes, one error middleware, and process-level crash handling."
---

# 05 · Error Handling

A backend spends most of its interesting moments handling things going wrong: invalid
input, missing records, a database that times out, a bug in your code. Good error
handling has three goals:

1. **The client** gets an accurate status code and a useful, safe message.
2. **You** get a log entry with enough context to fix the problem.
3. **The process** stays healthy — or, if it can't, dies quickly and gets restarted.

## Two kinds of errors

This distinction, popularized in the Node community, drives every decision below.

**Operational errors** are expected failures of a correct program: the user sent bad
JSON, the record doesn't exist, the email is taken, the database connection dropped, an
upstream API returned 503. You **handle** these: return a 4xx, retry, or degrade.

**Programmer errors** are bugs: `TypeError: Cannot read properties of undefined`, calling
a function with the wrong arguments, forgetting to `await`. You can't sensibly "handle" a
bug at runtime. Return a 500 for that request, log loudly, and fix the code. If the bug
happens outside any request (in a timer or event listener), the safest thing is to let
the process crash and be restarted, because its state may now be inconsistent.

## Error classes that carry an HTTP status

```js title="src/errors.js"
export class HttpError extends Error {
  constructor(status, message, details) {
    super(message);
    this.name = 'HttpError';
    this.status = status;
    this.details = details;
  }
}

export const badRequest = (msg, details) => new HttpError(400, msg, details);
export const unauthorized = (msg = 'authentication required') => new HttpError(401, msg);
export const notFound = (msg = 'not found') => new HttpError(404, msg);
export const conflict = (msg) => new HttpError(409, msg);

// Express 5: registered last, with four parameters
export function errorHandler(err, req, res, next) {
  if (res.headersSent) return next(err);

  // body-parser errors (malformed JSON, too large) carry a status and `expose`
  if (err.type === 'entity.parse.failed') err = badRequest('malformed JSON body');
  const status = err.status ?? 500;

  if (status >= 500) req.log.error({ err }, 'unhandled error');
  else req.log.warn({ status, msg: err.message }, 'request failed');

  res.status(status).json({
    error: status >= 500 ? 'internal server error' : err.message,
    ...(err.details && { details: err.details }),
  });
}
```

Route handlers now read naturally — they `throw` and move on:

```js
router.get('/:id', validate({ params: idParams }), async (req, res) => {
  const task = await findTask(req.valid.params.id, req.user.id);
  if (!task) throw notFound('task not found');
  res.json(task);
});
```

Registration order at the end of `createApp`:

```js
app.use((req, res, next) => next(notFound(`no route for ${req.method} ${req.path}`)));
app.use(errorHandler);
```

The catch-all 404 comes after every router; the error handler comes last of all.

### What the error handler is doing

- `res.headersSent` — if a response is already streaming, you can't send a new status.
  Delegating to Express's default handler closes the connection.
- Body-parser errors are normalized. `express.json()` throws an error with
  `type: 'entity.parse.failed'` (status 400) for malformed JSON and
  `'entity.too.large'` (413) for oversized bodies. Both already carry a `status`.
- **5xx responses never include `err.message`.** Internal messages can contain SQL,
  file paths, or hostnames. The client sees `internal server error`; the log gets the
  full error with stack.
- 4xx errors are logged at `warn`, 5xx at `error` — so alerts can key on the latter.

## Translating lower-level errors

Driver and library errors should be translated at the boundary where you know what they
mean. PostgreSQL reports a unique-constraint violation with SQLSTATE code `23505`; the
project's register route turns that into a 409:

```js
try {
  const user = await db.insertInto('users').values({ email, password_hash })
    .returning(['id', 'email']).executeTakeFirstOrThrow();
  res.status(201).json({ user, token: signToken(user, config.JWT_SECRET) });
} catch (err) {
  if (err.code === '23505') throw conflict('email already registered');
  throw err;   // anything else is unexpected → 500
}
```

When wrapping, keep the original with `cause` so the log still shows the root problem:

```js
throw new Error('failed to load exchange rates', { cause: err });
```

## Worked example: where errors can escape

```js
// 1. Sync throw in a handler → Express catches it → error middleware. ✅
app.get('/a', (req, res) => { JSON.parse('{'); });

// 2. Rejected promise from an async handler → Express 5 catches it. ✅
app.get('/b', async (req, res) => { await db.query('bad sql'); });

// 3. Throw inside a callback that runs later → nobody catches it. ❌ process crashes
app.get('/c', (req, res) => {
  setTimeout(() => { throw new Error('too late'); }, 10);
  res.send('ok');
});

// 4. A promise you started but didn't await → unhandled rejection. ❌ process crashes
app.get('/d', (req, res) => {
  sendWelcomeEmail(req.body.email);    // returns a promise; nobody handles its rejection
  res.status(202).end();
});
```

Case 3 and 4 escape because by the time they fail, Express's `try/catch` and promise
handler have long finished. Fix 4 with `sendWelcomeEmail(...).catch(err =>
req.log.error({ err }, 'welcome email failed'))`, or better, hand it to a job queue
(Level 4).

## Process-level handlers

For errors that escape everything:

```js title="src/server.js (excerpt)"
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'uncaught exception — exiting');
  process.exit(1);
});
process.on('unhandledRejection', (reason) => {
  logger.fatal({ err: reason }, 'unhandled rejection — exiting');
  process.exit(1);
});
```

These handlers exist to **log and exit**, not to keep running. Something like Docker,
Kubernetes, or systemd restarts the process. Level 4 lesson 09 refines this into a
graceful shutdown that finishes in-flight requests first.

## How It Actually Works

When Express invokes a handler, it wraps the call in `try/catch`; a synchronous throw
becomes `next(err)`. In Express 5, if the handler returns a thenable, the router also
attaches a rejection handler that calls `next(err)`. From then on, the router is in
error mode: it skips normal middleware and calls the next function with arity 4.

Asynchronous callbacks (`setTimeout`, event listeners, `fs` callbacks) run from a fresh
call stack started by the event loop. There is no `try` block above them — Express's
frame is gone. An exception there propagates to the top of the stack, where Node emits
`'uncaughtException'` on `process`; with no listener, it prints the stack and exits with
code 1.

Promises that reject without a handler are tracked by V8. After the microtask queue
drains, Node checks the list of rejected-but-unhandled promises and emits
`'unhandledRejection'`. The default mode (`--unhandled-rejections=throw`) raises it as an
uncaught exception, which is why case 4 crashes the process.

Why exit on an uncaught exception instead of continuing? The throw may have happened
halfway through updating shared state — a half-written cache entry, a connection
returned to the pool mid-transaction, a lock never released. Continuing means serving
future requests from a process in an unknown state.

## Common mistakes

- **Sending `err.message` or stacks for 500s** to clients.
- **Swallowing errors** (`catch {}`) or logging and then returning 200.
- **Checking `err.message` strings** to decide behavior; use `code`, `name`, or
  `instanceof`.
- **Fire-and-forget promises** without `.catch`.
- **Error middleware with three parameters** — it won't be called for errors.
- **Keeping the process alive after `uncaughtException`.**

## Exercise

1. Add a `ForbiddenError` (403) and use it for a route only admins may call.
2. Write tests for the error handler: malformed JSON → 400, unknown route → 404 JSON,
   a route that throws a `TypeError` → 500 with body `{ "error": "internal server error" }`.
3. Reproduce case 4 above in a small app. Observe the crash, then fix it with `.catch`.
4. Translate PostgreSQL foreign-key violations (SQLSTATE `23503`) into a 400 with a helpful
   message, e.g. when creating a task for a project id that doesn't exist.
