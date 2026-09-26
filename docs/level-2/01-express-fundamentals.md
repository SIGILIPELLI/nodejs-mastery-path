---
description: "Express 5 fundamentals: what it adds over node:http, the app object, req and res helpers, JSON parsing, and structuring a createApp factory."
---

# 01 · Express Fundamentals

In Level 1 you wrote an API on bare `node:http` and did the routing, body parsing, and
response formatting yourself. **Express** is the most widely used Node web framework
precisely because it does those three things and not much else. It is small, stable,
and unopinionated — which also means structure is your responsibility.

This course uses **Express 5**, the current major. If you read older tutorials, the
biggest differences are: rejected promises from async handlers are forwarded to error
handlers automatically, the path-matching syntax is stricter, and `req.body` is
`undefined` (not `{}`) when no body parser ran.

```bash
npm install express
```

## Hello, Express

```js title="app.js"
import express from 'express';

const app = express();
app.use(express.json());                     // parse JSON bodies into req.body

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });                // sets content-type and serializes
});

app.post('/echo', (req, res) => {
  res.status(201).json({ youSent: req.body });
});

app.listen(3000, () => console.log('http://localhost:3000'));
```

Compare with Level 1: `res.json` replaces the `sendJson` helper, `express.json()`
replaces `readJsonBody` (with a default 100 KB limit), and `app.get(path, fn)` replaces
the `if` chain.

## The objects you work with

`req` and `res` are still Node's `IncomingMessage` and `ServerResponse`; Express adds
properties and methods to them.

| On `req` | Meaning |
|---------|---------|
| `req.params` | route parameters (`/books/:id` → `{ id: '42' }`), always strings |
| `req.query` | parsed query string (`?tag=a&tag=b` → `{ tag: ['a', 'b'] }`) |
| `req.body` | parsed body — only if a body-parsing middleware ran |
| `req.get('authorization')` | case-insensitive header lookup |
| `req.path`, `req.originalUrl`, `req.ip`, `req.method` | request metadata |

| On `res` | Meaning |
|---------|---------|
| `res.status(201)` | set status (chainable) |
| `res.json(obj)` | send JSON |
| `res.send(x)` | send string/Buffer/object with a guessed content type |
| `res.set(name, value)`, `res.location(url)` | headers |
| `res.redirect(302, url)` | redirect |
| `res.sendFile(absPath)` | stream a file with proper headers |
| `res.end()` | finish with no body (e.g. after `res.status(204)`) |

## Worked example: observing what Express parses

This script mounts a few routes and drives them with Supertest (an HTTP testing
library; lesson 08 covers it) so you can see the parsed values without a browser:

```js title="explore.js"
import express from 'express';
import request from 'supertest';

const app = express();
app.use(express.json());

app.use((req, res, next) => {
  const start = performance.now();
  res.on('finish', () => console.log(`${req.method} ${req.originalUrl} -> ${res.statusCode} (${(performance.now() - start).toFixed(1)} ms)`));
  next();
});

app.get('/books/:id', (req, res) => res.json({ params: req.params, query: req.query }));
app.post('/echo', (req, res) => res.json({ body: req.body }));
app.get('/boom', async () => { throw new Error('async failure'); });
app.use((err, req, res, next) => res.status(500).json({ caught: err.message }));

for (const [method, url, body] of [
  ['get', '/books/42?format=pdf&tag=a&tag=b'],
  ['post', '/echo', { x: 1 }],
  ['get', '/boom'],
  ['get', '/nope'],
]) {
  const r = await request(app)[method](url).send(body);
  console.log(r.status, JSON.stringify(r.body));
}
```

Output from running it:

```text
GET /books/42?format=pdf&tag=a&tag=b -> 200 (3.4 ms)
200 {"params":{"id":"42"},"query":{"format":"pdf","tag":["a","b"]}}
POST /echo -> 200 (0.2 ms)
200 {"body":{"x":1}}
GET /boom -> 500 (0.3 ms)
500 {"caught":"async failure"}
GET /nope -> 404 (0.6 ms)
404 {}
```

Notice:

- `id` is the **string** `'42'`. Convert and validate it yourself (lesson 04).
- A repeated query key becomes an **array**. Code expecting a string will break when a
  client sends `?tag=a&tag=b` — another reason to validate.
- The `async` handler threw, and Express 5 routed the rejection to the error
  middleware. (In Express 4 this would have been an unhandled rejection.)
- An unmatched route gets Express's default 404 (HTML, so the JSON body is empty). You
  will replace it with a JSON 404 in lesson 05.

## Structuring an app: the factory pattern

Don't create the app and call `listen` in the same module. Export a function that builds
the app from its dependencies, and listen in a separate entry file:

```js title="src/app.js"
import express from 'express';

export function createApp({ db, logger }) {
  const app = express();
  app.disable('x-powered-by');       // don't advertise the framework
  app.use(express.json({ limit: '100kb' }));
  app.get('/health', (req, res) => res.json({ status: 'ok' }));
  // app.use('/tasks', tasksRouter({ db }));  — lesson 02
  return app;
}
```

```js title="src/server.js"
import { createApp } from './app.js';
const app = createApp({ db: /* real pool */ null, logger: console });
app.listen(process.env.PORT ?? 3000);
```

Tests import `createApp` and pass a test database; they never open a real port. The
Level 2 project uses exactly this layout.

## How It Actually Works

`express()` returns a function `app(req, res, next)` with methods attached. `app.listen`
is just `http.createServer(app).listen(...)` — the same server from Level 1 with Express
as the request handler.

Internally the app holds a **router**: an ordered list of *layers*. Each layer has a path
matcher and a handler. `app.use(fn)` adds a layer that matches any path prefix;
`app.get('/x', fn)` adds a route layer that matches the exact path and method. For each
request, Express walks the list from the top, and for each matching layer calls its
handler with a `next` function. Calling `next()` continues the walk; sending a response
without calling `next()` ends it. If the walk reaches the end, Express's final handler
sends a 404.

Before your code sees them, Express sets the prototypes of `req` and `res` to its own
`request`/`response` objects, which inherit from Node's classes. That is how `res.json`
and `req.get` appear on the plain Node objects.

`res.json(obj)` calls `JSON.stringify` (honoring `app.set('json spaces')` and replacer
settings), sets `Content-Type: application/json; charset=utf-8` if unset, computes
`Content-Length`, and calls `res.end`. For GET/HEAD it also generates an **ETag** and
answers `304 Not Modified` if the client's `If-None-Match` matches.

Express 5's automatic async error handling works because the router checks whether a
handler returned a promise and attaches a `.catch(next)`.

## Common mistakes

- **Forgetting `express.json()`** → `req.body` is `undefined`.
- **Sending two responses** (`res.json` then falling through to another `res.json`) →
  `ERR_HTTP_HEADERS_SENT`. `return res.json(...)` in branches.
- **Assuming `req.params.id` is a number.**
- **Registering middleware after routes** that need it; order matters (next lesson).
- **Building one giant `app.js`** instead of routers per resource.

## Exercise

1. Port the Level 1 notes API to Express, keeping identical behavior and status codes.
   Count the lines saved.
2. Add a route `GET /files/*filepath` (Express 5 wildcard syntax) and log what
   `req.params.filepath` contains for `/files/a/b/c.txt`.
3. Send a `text/plain` body to `POST /echo`. What is `req.body`, and why? Add
   `express.text()` and try again.
4. Call the same `GET /health` twice with `curl -i`, copy the ETag, and send it back with
   `-H 'If-None-Match: <etag>'`. Explain the 304.
