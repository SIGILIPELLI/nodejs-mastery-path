---
description: "Build a JSON API with only node:http: requests and responses as streams, routing by hand, body parsing with size limits, status codes, and keep-alive."
---

# 08 · A Raw HTTP Server with node:http

Frameworks like Express (Level 2) are thin layers over `node:http`. Building a small API
without one first shows you exactly what the framework does for you — and what it does
not. You will parse URLs, read request bodies, set headers, and route by hand.

## The smallest server

```js
import { createServer } from 'node:http';

createServer((req, res) => {
  res.writeHead(200, { 'content-type': 'text/plain' });
  res.end('hello\n');
}).listen(3000);
```

For every request, Node calls your function with two objects:

- `req` — an `http.IncomingMessage`: `req.method`, `req.url` (path + query string only),
  `req.headers` (lowercased names), and the body as a **readable stream**.
- `res` — an `http.ServerResponse`: `res.statusCode`, `res.setHeader()`, `res.writeHead()`,
  `res.write()`, and `res.end()`. It is a **writable stream**.

Forgetting `res.end()` leaves the client hanging until it times out.

## Worked example: a notes API with no dependencies

```js title="http-server.mjs"
import { createServer } from 'node:http';

const notes = new Map();
let nextId = 1;
const MAX_BODY = 10_000; // bytes

function sendJson(res, status, data) {
  const body = JSON.stringify(data);
  res.writeHead(status, {
    'content-type': 'application/json; charset=utf-8',
    'content-length': Buffer.byteLength(body),
  });
  res.end(body);
}

async function readJsonBody(req) {
  let size = 0;
  const chunks = [];
  for await (const chunk of req) {
    size += chunk.length;
    if (size > MAX_BODY) throw Object.assign(new Error('body too large'), { status: 413 });
    chunks.push(chunk);
  }
  try {
    return JSON.parse(Buffer.concat(chunks).toString('utf8'));
  } catch {
    throw Object.assign(new Error('invalid JSON'), { status: 400 });
  }
}

const server = createServer(async (req, res) => {
  const url = new URL(req.url, `http://${req.headers.host ?? 'localhost'}`);
  const match = url.pathname.match(/^\/notes\/(\d+)$/);

  try {
    if (req.method === 'GET' && url.pathname === '/notes') {
      const q = url.searchParams.get('q')?.toLowerCase();
      const list = [...notes.values()].filter(n => !q || n.text.toLowerCase().includes(q));
      return sendJson(res, 200, list);
    }
    if (req.method === 'POST' && url.pathname === '/notes') {
      const { text } = await readJsonBody(req);
      if (typeof text !== 'string' || !text.trim()) return sendJson(res, 400, { error: 'text is required' });
      const note = { id: nextId++, text: text.trim() };
      notes.set(note.id, note);
      res.setHeader('location', `/notes/${note.id}`);
      return sendJson(res, 201, note);
    }
    if (req.method === 'GET' && match) {
      const note = notes.get(Number(match[1]));
      return note ? sendJson(res, 200, note) : sendJson(res, 404, { error: 'not found' });
    }
    if (req.method === 'DELETE' && match) {
      const existed = notes.delete(Number(match[1]));
      res.writeHead(existed ? 204 : 404).end();
      return;
    }
    sendJson(res, 404, { error: `no route for ${req.method} ${url.pathname}` });
  } catch (err) {
    sendJson(res, err.status ?? 500, { error: err.status ? err.message : 'internal error' });
    if (!err.status) console.error(err);
  }
});

server.listen(3000, () => console.log('notes API on http://localhost:3000'));
```

Exercising it with `curl`:

```bash
curl -i -X POST localhost:3000/notes -H 'content-type: application/json' -d '{"text":"buy milk"}'
```

```text
HTTP/1.1 201 Created
location: /notes/1
content-type: application/json; charset=utf-8
content-length: 26
Date: ...
Connection: keep-alive
Keep-Alive: timeout=5

{"id":1,"text":"buy milk"}
```

Then, after adding a second note:

```text
$ curl 'localhost:3000/notes?q=milk'
[{"id":1,"text":"buy milk"}]
$ curl -X POST localhost:3000/notes -d '{oops'
{"error":"invalid JSON"}
$ curl -o /dev/null -w "%{http_code}\n" -X DELETE localhost:3000/notes/1
204
$ curl localhost:3000/notes/1
{"error":"not found"}
$ curl localhost:3000/users
{"error":"no route for GET /users"}
```

### What this code is doing that a framework would do for you

- **URL parsing.** `req.url` is only `/notes?q=milk`; `new URL()` needs a base to parse
  it. `searchParams` handles decoding.
- **Routing.** A chain of `if`s with a regex for path parameters. It works for five
  routes; it becomes painful at fifty.
- **Body parsing.** The body arrives in chunks. We collect them, enforce a size limit
  (without one, a client can send gigabytes and exhaust memory), and parse JSON.
- **Consistent responses.** `sendJson` sets `content-type` and `content-length`.
- **Error mapping.** Known errors carry a `status`; unknown ones become a generic 500
  without leaking internals to the client.

## How It Actually Works

When a TCP connection arrives, libuv's poll phase reports the listening socket as
readable; Node `accept`s it and creates a `net.Socket`. The HTTP server attaches
**llhttp**, a C parser, to that socket. As bytes arrive, llhttp parses the request line
and headers; once headers are complete, Node creates `req` and `res` and emits
`'request'` — calling your handler. The body has not necessarily arrived yet; body bytes
are pushed into `req` as they are parsed, which is why the body is a stream.

On the response side, the first `res.write`/`res.end` serializes the status line and
headers. If you did not set `content-length`, Node uses
`Transfer-Encoding: chunked`, framing each write with its size, so the client knows
where the body ends without knowing the total length up front.

**Keep-alive.** HTTP/1.1 connections are persistent by default: after the response, the
socket stays open for the next request (Node closes idle ones after
`server.keepAliveTimeout`, 5 seconds by default — hence `Keep-Alive: timeout=5`). Each
connection handles requests one at a time, but many connections are served concurrently
by the single event loop, since each is just a socket in the poller.

**Timeouts that protect you.** `server.headersTimeout` and `server.requestTimeout` cap how
long a client may take to send headers and the full request, defending against
slow-loris style attacks that hold connections open by trickling bytes.

## Common mistakes

- **Not ending the response** on some code path (often an error path).
- **Writing after `end()`** or calling `writeHead` twice → `ERR_HTTP_HEADERS_SENT`.
- **No body size limit.** Always cap it.
- **Trusting `content-type`** — clients lie. Parse defensively, as `readJsonBody` does.
- **Leaking stack traces** in 500 responses. Log them server-side; send a generic
  message.
- **`JSON.stringify` of huge arrays** in a handler blocks the event loop; paginate.

## Exercise

1. Add `PUT /notes/:id` that replaces a note's text, returning 404 for unknown ids and 400
   for invalid bodies.
2. Reject `POST` requests whose `content-type` is not `application/json` with **415**.
3. Add a `GET /health` route that returns `{ status: "ok", uptime: process.uptime() }`.
4. Send a 20 KB body with `curl --data-binary @bigfile.json` and confirm the 413. Then use
   `curl -v` to see the `Connection` header and observe keep-alive by making two requests
   in one `curl` command.
