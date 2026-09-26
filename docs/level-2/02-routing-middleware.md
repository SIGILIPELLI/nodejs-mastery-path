---
description: "Express routing and middleware: route paths and params, Router modules, middleware order, next(), route-level guards, and writing your own middleware."
---

# 02 · Routing & Middleware

Everything in Express is either a **route handler** (produces a response) or
**middleware** (does something to the request, then passes control on). In fact they are
the same shape — `(req, res, next)` — and the difference is only whether the function
calls `next()`. Once that clicks, authentication, logging, CORS, rate limiting, and body
parsing all look the same: functions in a pipeline.

## Route paths in Express 5

```js
app.get('/books', list);                  // exact path
app.get('/books/:id', getOne);            // named parameter → req.params.id
app.get('/users/:userId/books/:bookId', h);
app.get('/docs{/:section}', docs);        // optional segment: /docs and /docs/intro
app.get('/files/*filepath', files);       // wildcard, must be named; value is an array of segments
app.route('/books/:id')                   // several methods, one path
  .get(getOne)
  .patch(update)
  .delete(remove);
```

Express 5 uses a newer version of `path-to-regexp` than Express 4. Regex characters like
`?`, `+`, and `(` are no longer special inside path strings, and unnamed `*` wildcards
are not allowed. Old tutorials using `'/files/*'` or `'/:id?'` need updating to the
forms above.

## Routers: one module per resource

`express.Router()` is a mini-app with its own middleware and routes, mounted at a prefix:

```js title="src/routes/books.js"
import { Router } from 'express';

export function booksRouter({ db }) {
  const router = Router();
  router.get('/', async (req, res) => res.json(await db.listBooks()));
  router.get('/:id', async (req, res) => { /* ... */ });
  return router;
}
```

```js title="src/app.js"
app.use('/books', booksRouter({ db }));   // router sees paths relative to /books
```

Inside the router, `req.baseUrl` is `/books` and `req.path` is the rest. Passing
dependencies (`db`) into a function that builds the router, rather than importing a
global, keeps routers testable.

## Middleware and order

Middleware runs in the order it is registered, and only for requests whose path matches
its mount point:

```js
app.use(helmet());                 // 1. security headers for everything
app.use(express.json());           // 2. body parsing
app.use(requestLogger);            // 3. logging
app.use('/admin', requireAdmin);   // 4. only for /admin/*
app.use('/books', booksRouter);    // 5. routes
app.use(notFoundHandler);          // 6. nothing matched
app.use(errorHandler);             // 7. four-argument error middleware, last
```

If `requestLogger` were registered after the routers, it would never see requests the
routers already answered.

## Worked example: tracing the pipeline

Middleware can be global, router-level, or route-level (listed between the path and the
handler). This example records which functions ran:

```js title="pipeline.js"
import express, { Router } from 'express';
import request from 'supertest';

const trace = (name) => (req, res, next) => { req.trail.push(name); next(); };

function requireApiKey(req, res, next) {
  if (req.get('x-api-key') !== 'dev-key') {
    return res.status(401).json({ error: 'missing or invalid API key' });
  }
  next();
}

const books = Router();
books.use(trace('books-router'));
books.get('/', trace('list-handler'), (req, res) => res.json(req.trail));
books.post('/', requireApiKey, trace('create-handler'), (req, res) => res.status(201).json(req.trail));

const app = express();
app.use((req, res, next) => { req.trail = ['app']; next(); });
app.use('/books', books);

console.log((await request(app).get('/books')).body);
console.log((await request(app).post('/books')).status);
console.log((await request(app).post('/books').set('x-api-key', 'dev-key')).body);
```

```text
[ 'app', 'books-router', 'list-handler' ]
401
[ 'app', 'books-router', 'create-handler' ]
```

`requireApiKey` is a **guard**: on failure it responds and does not call `next()`, so the
handler never runs. This is exactly how authentication middleware works in lesson 07.

## Writing useful middleware

**Timing header:**

```js
export function responseTime(req, res, next) {
  const start = process.hrtime.bigint();
  res.on('finish', () => {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    req.log?.info({ ms }, 'request finished');
  });
  next();
}
```

Headers can't be set after the response starts, so work that needs the final status
goes in a `'finish'` listener.

**Configurable middleware** is a function that returns middleware:

```js
export function requireRole(role) {
  return (req, res, next) => {
    if (req.user?.role !== role) return res.status(403).json({ error: 'forbidden' });
    next();
  };
}

router.delete('/:id', requireRole('admin'), removeBook);
```

**Attaching data for later handlers** — set a property on `req` (e.g. `req.user`,
`req.valid`). Keep it namespaced to avoid collisions with Express's own properties.

**CORS** — if browsers on another origin call your API, use the `cors` package with an
explicit allowlist rather than `*` for authenticated APIs:
`app.use(cors({ origin: ['https://app.example.com'], credentials: true }))`.

## How It Actually Works

Each `app.use`/`router.get` call appends a **Layer** to a stack. A Layer stores a
compiled path matcher (from `path-to-regexp`), whether it is an end-point (`route`) or a
prefix (`use`), and the handler. When a request comes in, the router runs an internal
`next()` function that advances an index through the stack:

1. Find the next layer whose matcher accepts `req.path` (and method, for routes).
2. For `use` layers, temporarily strip the matched prefix from `req.url` and set
   `req.baseUrl`, so nested routers see relative paths; restore it afterwards.
3. Call the handler with `(req, res, next)`, where `next` is this same advancing
   function.
4. If `next(err)` is called with an argument, switch to **error mode**: skip every layer
   whose handler has fewer than four parameters until reaching an `(err, req, res, next)`
   handler. Express detects error handlers by `fn.length === 4`.
5. `next('route')` skips the remaining handlers of the current route; `next('router')`
   exits the current router.

A mounted `Router` is itself a handler function that runs its own stack and calls the
outer `next` when it runs out — which is how unmatched paths fall through a router to the
404 handler.

Everything happens by function calls in the same tick; nothing is queued. Middleware that
does I/O simply calls `next()` later, from its callback or after an `await`.

## Common mistakes

- **Forgetting to call `next()`** (and not responding) → the request hangs until the
  client times out.
- **Calling `next()` after responding** → later handlers try to respond again.
- **Wrong order** — auth registered after the routes it should protect.
- **Error middleware with three parameters** — Express treats it as normal middleware
  and it never receives errors.
- **Overlapping routes** — `/books/:id` registered before `/books/search` captures
  `search` as an id. Put specific routes first or constrain parameters.
- **Doing heavy synchronous work in global middleware** — it runs on every request.

## Exercise

1. Split your Express notes API into `routes/notes.js` exporting `notesRouter({ store })`.
2. Write `requestId()` middleware that reads `x-request-id` or generates one with
   `crypto.randomUUID()`, sets it on `req.id` and on the response header.
3. Write `requireRole(role)` and protect `DELETE /notes/:id` with it using a fake
   `req.user` set by a previous middleware from an `x-role` header.
4. Register `/notes/:id` before `/notes/stats` and observe the bug; fix it two ways
   (reordering, and constraining the id with validation that calls `next('route')`).
