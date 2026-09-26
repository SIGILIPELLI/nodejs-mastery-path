---
description: "Project: a real-time chat server with Express and ws: token-authenticated upgrades, rooms, history, presence, rate limits, heartbeats, and WebSocket tests."
---

# 10 · Project — A Real-Time Chat App

This project combines Level 3 into a working multi-room chat: an Express app that issues
short-lived tokens and serves a small browser client, and a WebSocket server that
authenticates connections during the upgrade, validates every message, keeps bounded
per-room history, rate-limits each connection, detects dead peers, and shuts down
cleanly. It's tested end to end with Node's built-in `WebSocket` client.

## Features and the lesson each comes from

| Feature | Lesson |
|---------|--------|
| Upgrade-time auth with a signed, expiring token and Origin check | 07 WebSockets, 03 Buffers (HMAC bytes) |
| Messages validated with a Zod discriminated union | L2-04 |
| Rooms, presence, typing, bounded history | 07 |
| Slow-consumer protection via `bufferedAmount` | 02 Streams & backpressure |
| Per-connection token-bucket rate limit | 09 Security |
| Heartbeats with ping/pong | 07 |
| Graceful shutdown with close code 1001 | preview of L4-09 |

## Layout

```text
chat/
  src/protocol.js   # message schemas
  src/rooms.js      # membership, history, broadcast
  src/tokens.js     # signed expiring tokens
  src/server.js     # createChatServer(): Express + ws
  src/main.js       # entry point
  public/index.html # minimal browser client
  test/chat.test.js
```

Dependencies: `express`, `ws`, `zod`.

## The protocol

```js title="src/protocol.js"
import { z } from 'zod';

const room = z.string().regex(/^[a-z0-9-]{1,32}$/, 'room names are lowercase letters, digits, dashes');

// Every message a client may send, discriminated by `type`
export const ClientMessage = z.discriminatedUnion('type', [
  z.object({ type: z.literal('join'), room }),
  z.object({ type: z.literal('leave'), room }),
  z.object({ type: z.literal('say'), room, text: z.string().trim().min(1).max(1000) }),
  z.object({ type: z.literal('typing'), room }),
]);
```

A single schema describes every message a client may send. Anything else — wrong type,
missing room, a 5 KB "text", a room name with spaces — is rejected before it reaches the
room logic.

## Room state

```js title="src/rooms.js"
// In-memory room state for ONE process. Scaling out requires pub/sub (see lesson text).
export class Rooms {
  #members = new Map();   // room -> Set<socket>
  #history = new Map();   // room -> recent messages (bounded)
  constructor({ historySize = 50 } = {}) { this.historySize = historySize; }

  join(room, socket) {
    if (!this.#members.has(room)) this.#members.set(room, new Set());
    this.#members.get(room).add(socket);
    return this.#history.get(room) ?? [];
  }

  leave(room, socket) {
    const set = this.#members.get(room);
    if (!set) return;
    set.delete(socket);
    if (set.size === 0) this.#members.delete(room);
  }

  leaveAll(socket) {
    for (const room of [...this.#members.keys()]) this.leave(room, socket);
  }

  isMember(room, socket) { return this.#members.get(room)?.has(socket) ?? false; }

  record(room, message) {
    const list = this.#history.get(room) ?? [];
    list.push(message);
    if (list.length > this.historySize) list.shift();
    this.#history.set(room, list);
  }

  broadcast(room, payload, { except } = {}) {
    const data = JSON.stringify(payload);          // serialize once
    for (const socket of this.#members.get(room) ?? []) {
      if (socket === except || socket.readyState !== socket.OPEN) continue;
      if (socket.bufferedAmount > 1_000_000) { socket.terminate(); continue; }  // slow consumer
      socket.send(data);
    }
  }

  count(room) { return this.#members.get(room)?.size ?? 0; }
}
```

`broadcast` serializes once, skips sockets that are closing, and terminates any client
with more than ~1 MB queued — a slow consumer must not be allowed to grow server memory
without bound.

## Tokens

```js title="src/tokens.js"
import { createHmac, timingSafeEqual } from 'node:crypto';

// Tiny signed, expiring token: base64url(json).base64url(hmac). Enough for a demo;
// in the Level 2 API you would reuse the JWT helpers instead.
export function issueToken(name, secret, ttlSeconds = 60) {
  const body = Buffer.from(JSON.stringify({ name, exp: Date.now() + ttlSeconds * 1000 })).toString('base64url');
  const sig = createHmac('sha256', secret).update(body).digest('base64url');
  return `${body}.${sig}`;
}

export function verifyToken(token, secret) {
  const [body, sig] = String(token).split('.');
  if (!body || !sig) return null;
  const expected = createHmac('sha256', secret).update(body).digest();
  const given = Buffer.from(sig, 'base64url');
  if (given.length !== expected.length || !timingSafeEqual(given, expected)) return null;
  const payload = JSON.parse(Buffer.from(body, 'base64url').toString());
  return payload.exp > Date.now() ? payload : null;
}
```

## The server

```js title="src/server.js"
import express from 'express';
import { createServer } from 'node:http';
import { WebSocketServer } from 'ws';
import { z } from 'zod';
import { ClientMessage } from './protocol.js';
import { Rooms } from './rooms.js';
import { issueToken, verifyToken } from './tokens.js';

export function createChatServer({ secret, allowedOrigins = [], heartbeatMs = 30_000 }) {
  const app = express();
  app.use(express.json({ limit: '10kb' }));
  app.use(express.static(new URL('../public', import.meta.url).pathname));

  // "Login": pick a display name, get a short-lived token for the WebSocket upgrade
  const Login = z.object({ name: z.string().trim().regex(/^[\w-]{2,20}$/) });
  app.post('/session', (req, res) => {
    const parsed = Login.safeParse(req.body);
    if (!parsed.success) return res.status(400).json({ error: 'name must be 2-20 letters, digits, _ or -' });
    res.json({ token: issueToken(parsed.data.name, secret) });
  });

  const server = createServer(app);
  const wss = new WebSocketServer({ noServer: true, maxPayload: 16 * 1024 });
  const rooms = new Rooms();

  server.on('upgrade', (req, socket, head) => {
    const url = new URL(req.url, 'http://localhost');
    const origin = req.headers.origin;
    const user = url.pathname === '/ws' ? verifyToken(url.searchParams.get('token'), secret) : null;
    if (!user || (origin && allowedOrigins.length && !allowedOrigins.includes(origin))) {
      socket.write('HTTP/1.1 401 Unauthorized\r\nConnection: close\r\n\r\n');
      return socket.destroy();
    }
    wss.handleUpgrade(req, socket, head, (ws) => wss.emit('connection', ws, user));
  });

  wss.on('connection', (ws, user) => {
    ws.user = user;
    ws.isAlive = true;
    ws.tokens = 10;                                  // simple per-socket token bucket
    ws.on('pong', () => { ws.isAlive = true; });
    ws.on('error', () => {});                        // 'close' follows; nothing else to do
    ws.on('close', () => rooms.leaveAll(ws));
    const send = (payload) => ws.send(JSON.stringify(payload));

    ws.on('message', (data, isBinary) => {
      if (isBinary) return ws.close(1003, 'text frames only');
      if (ws.tokens <= 0) return send({ type: 'error', error: 'slow down' });
      ws.tokens--;

      let msg;
      try { msg = ClientMessage.parse(JSON.parse(data.toString())); }
      catch { return send({ type: 'error', error: 'invalid message' }); }

      switch (msg.type) {
        case 'join': {
          const history = rooms.join(msg.room, ws);
          send({ type: 'joined', room: msg.room, history, online: rooms.count(msg.room) });
          rooms.broadcast(msg.room, { type: 'presence', room: msg.room, user: user.name, event: 'join' }, { except: ws });
          break;
        }
        case 'leave':
          rooms.leave(msg.room, ws);
          rooms.broadcast(msg.room, { type: 'presence', room: msg.room, user: user.name, event: 'leave' });
          break;
        case 'say': {
          if (!rooms.isMember(msg.room, ws)) return send({ type: 'error', error: `join ${msg.room} first` });
          const message = { type: 'message', room: msg.room, user: user.name, text: msg.text, at: new Date().toISOString() };
          rooms.record(msg.room, message);
          rooms.broadcast(msg.room, message);
          break;
        }
        case 'typing':
          if (rooms.isMember(msg.room, ws)) rooms.broadcast(msg.room, { type: 'typing', room: msg.room, user: user.name }, { except: ws });
          break;
      }
    });
  });

  // Refill rate-limit tokens and detect dead connections
  const refill = setInterval(() => { for (const ws of wss.clients) ws.tokens = Math.min(10, ws.tokens + 5); }, 1000);
  const heartbeat = setInterval(() => {
    for (const ws of wss.clients) {
      if (!ws.isAlive) { ws.terminate(); continue; }
      ws.isAlive = false;
      ws.ping();
    }
  }, heartbeatMs);

  function close() {
    clearInterval(refill);
    clearInterval(heartbeat);
    for (const ws of wss.clients) ws.close(1001, 'server shutting down');
    return new Promise((resolve) => server.close(resolve));
  }

  return { server, close };
}
```

```js title="src/main.js"
import { createChatServer } from './server.js';

const secret = process.env.CHAT_SECRET;
if (!secret || secret.length < 32) throw new Error('CHAT_SECRET (32+ chars) is required');

const { server, close } = createChatServer({
  secret,
  allowedOrigins: (process.env.ALLOWED_ORIGINS ?? 'http://localhost:3400').split(','),
});
server.listen(Number(process.env.PORT ?? 3400), () => console.log('chat on http://localhost:3400'));

process.on('SIGTERM', async () => { await close(); process.exit(0); });
process.on('SIGINT', async () => { await close(); process.exit(0); });
```

Why `noServer: true` and a manual `'upgrade'` handler? It lets us **reject** bad
connections with a proper HTTP 401 *before* they become WebSockets, instead of accepting
them and closing immediately.

## A minimal browser client

```html title="public/index.html"
<!doctype html>
<html lang="en">
<meta charset="utf-8">
<title>Node chat</title>
<style>
  body { font: 16px system-ui, sans-serif; max-width: 40rem; margin: 2rem auto; padding: 0 1rem; }
  #log { border: 1px solid #ccc; height: 20rem; overflow-y: auto; padding: .5rem; }
  .sys { color: #777; font-style: italic; }
</style>
<form id="login"><input id="name" placeholder="your name" required> <button>Join #general</button></form>
<div id="log" hidden></div>
<form id="say" hidden><input id="text" autocomplete="off" required> <button>Send</button></form>
<script type="module">
  const $ = (id) => document.getElementById(id);
  const line = (text, cls) => {
    const p = document.createElement('p');
    p.textContent = text;                     // textContent, never innerHTML: no XSS
    if (cls) p.className = cls;
    $('log').append(p);
    $('log').scrollTop = $('log').scrollHeight;
  };

  $('login').onsubmit = async (e) => {
    e.preventDefault();
    const res = await fetch('/session', { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ name: $('name').value }) });
    const { token, error } = await res.json();
    if (error) return alert(error);
    const ws = new WebSocket(`${location.origin.replace('http', 'ws')}/ws?token=${encodeURIComponent(token)}`);
    ws.onopen = () => { ws.send(JSON.stringify({ type: 'join', room: 'general' })); $('login').hidden = true; $('log').hidden = $('say').hidden = false; };
    ws.onmessage = (e) => {
      const m = JSON.parse(e.data);
      if (m.type === 'joined') { m.history.forEach((h) => line(`${h.user}: ${h.text}`)); line(`joined #${m.room} (${m.online} online)`, 'sys'); }
      if (m.type === 'message') line(`${m.user}: ${m.text}`);
      if (m.type === 'presence') line(`${m.user} ${m.event === 'join' ? 'joined' : 'left'}`, 'sys');
      if (m.type === 'error') line(`error: ${m.error}`, 'sys');
    };
    ws.onclose = () => line('disconnected', 'sys');
    $('say').onsubmit = (e) => { e.preventDefault(); ws.send(JSON.stringify({ type: 'say', room: 'general', text: $('text').value })); $('text').value = ''; };
  };
</script>
</html>
```

Run it with `CHAT_SECRET=<32+ random chars> node src/main.js`, open
`http://localhost:3400` in two browser windows, and chat. Starting it and checking the
basics from a terminal:

```text
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" localhost:3400/
200 text/html; charset=utf-8
$ curl -s -X POST localhost:3400/session -H 'content-type: application/json' -d '{"name":"x"}'
{"error":"name must be 2-20 letters, digits, _ or -"}
```

## Tests

```js title="test/chat.test.js"
import { test, before, after } from 'node:test';
import assert from 'node:assert/strict';
import { once } from 'node:events';
import { createChatServer } from '../src/server.js';

const secret = 'test-secret-with-at-least-32-characters!';
let chat, base;

before(async () => {
  chat = createChatServer({ secret });
  chat.server.listen(0);
  await once(chat.server, 'listening');
  base = `localhost:${chat.server.address().port}`;
});
after(() => chat.close());

async function login(name) {
  const res = await fetch(`http://${base}/session`, {
    method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ name }),
  });
  return (await res.json()).token;
}

// Connect and collect messages; next(type) resolves with the next message of that type
async function connect(name) {
  const ws = new WebSocket(`ws://${base}/ws?token=${await login(name)}`);
  const inbox = [];
  const waiters = [];
  ws.addEventListener('message', (e) => {
    const m = JSON.parse(e.data);
    const i = waiters.findIndex((w) => w.type === m.type);
    if (i >= 0) waiters.splice(i, 1)[0].resolve(m); else inbox.push(m);
  });
  await once(ws, 'open');
  ws.json = (obj) => ws.send(JSON.stringify(obj));
  ws.next = (type) => {
    const i = inbox.findIndex((m) => m.type === type);
    if (i >= 0) return Promise.resolve(inbox.splice(i, 1)[0]);
    return new Promise((resolve) => waiters.push({ type, resolve }));
  };
  return ws;
}

test('rejects upgrades without a valid token', async () => {
  const ws = new WebSocket(`ws://${base}/ws?token=forged.token`);
  const [event] = await once(ws, 'error');
  assert.equal(event.type, 'error');
});

test('messages reach everyone in the room, with history for late joiners', async () => {
  const alice = await connect('alice');
  const bob = await connect('bob');
  alice.json({ type: 'join', room: 'general' });
  await alice.next('joined');
  bob.json({ type: 'join', room: 'general' });
  const joined = await bob.next('joined');
  assert.equal(joined.online, 2);
  assert.equal((await alice.next('presence')).user, 'bob');

  alice.json({ type: 'say', room: 'general', text: '  hello bob  ' });
  const [a, b] = await Promise.all([alice.next('message'), bob.next('message')]);
  assert.equal(a.text, 'hello bob');
  assert.equal(b.user, 'alice');

  const carol = await connect('carol');
  carol.json({ type: 'join', room: 'general' });
  const late = await carol.next('joined');
  assert.deepEqual(late.history.map((m) => m.text), ['hello bob']);
  for (const ws of [alice, bob, carol]) ws.close();
});

test('invalid messages and non-members get errors, not crashes', async () => {
  const eve = await connect('eve');
  eve.send('not json');
  assert.equal((await eve.next('error')).error, 'invalid message');
  eve.json({ type: 'say', room: 'general', text: 'sneaky' });
  assert.equal((await eve.next('error')).error, 'join general first');
  eve.close();
});

test('per-connection rate limit', async () => {
  const spammer = await connect('spammer');
  spammer.json({ type: 'join', room: 'spam' });
  await spammer.next('joined');
  for (let i = 0; i < 15; i++) spammer.json({ type: 'say', room: 'spam', text: `msg ${i}` });
  assert.equal((await spammer.next('error')).error, 'slow down');
  spammer.close();
});
```

```text
$ node --test 'test/*.test.js'
✔ rejects upgrades without a valid token
✔ messages reach everyone in the room, with history for late joiners
✔ invalid messages and non-members get errors, not crashes
✔ per-connection rate limit
ℹ tests 4
ℹ pass 4
ℹ fail 0
```

The helper `connect()` buffers incoming messages and exposes `next(type)`, which turns
the event stream into awaitable steps — the same idea as `events.once`, filtered by
message type. Tests listen on port `0` so the OS picks a free port.

## Scaling out: what changes with several processes

Everything in `Rooms` lives in one process's memory. To run several instances behind a
load balancer:

1. **Fan-out via pub/sub.** When a process receives a `say`, it publishes
   `{ room, message }` to a Redis channel (e.g. `chat:general`). Every process subscribes
   and calls its local `rooms.broadcast` for messages it receives. A subscriber needs its
   own dedicated Redis connection.
2. **Shared history.** Store recent messages in a Redis list per room (`LPUSH` + `LTRIM`
   to bound it) or in Postgres, rather than in process memory.
3. **Presence** becomes eventually consistent: keep a Redis set per room with entries that
   expire unless refreshed by heartbeats.
4. **Load balancer settings.** WebSockets are long-lived; configure idle timeouts above
   your heartbeat interval and make sure the balancer passes `Upgrade` headers.

This needs a Redis server, so it's left as an exercise rather than shown with output.

## How It Actually Works

Follow one chat message:

1. Alice's browser sends a masked text frame over her TCP connection.
2. libuv's poll phase reports her socket readable; `ws` unmasks and assembles the frame,
   checks it against `maxPayload`, and emits `'message'` with a Buffer.
3. Our handler spends a rate-limit token, parses JSON, validates with Zod, and checks room
   membership — all synchronous, microseconds of work.
4. `rooms.broadcast` serializes once and calls `send` for each member. Each `send` frames
   the data and writes it to that member's socket. Writes that the kernel can't accept yet
   are queued in user space — visible as `bufferedAmount`.
5. The kernel transmits; each browser's `onmessage` fires.

No step waits on I/O, so a single process handles a lot of chat traffic; the costs grow
with (messages per second × members per room). The heartbeat timer, the token refill
timer, and the listening socket are the handles that keep the process alive; `close()`
clears the timers, sends close frames with code 1001 ("going away"), and closes the server
so the process can exit.

## Common mistakes in real-time projects

- Rendering messages with `innerHTML` (stored XSS through chat). The client uses
  `textContent`.
- Trusting client-sent usernames; here the name comes from the verified token.
- Unbounded history, per-room state that's never cleaned up, or sockets never removed
  from rooms on `close`.
- No rate limiting — one client can flood a room.
- Forgetting that timers keep the process alive, so tests hang.

## Exercise

1. Add direct messages: `{ type: "dm", to: "bob", text }`, delivered to all of Bob's
   connections (he may have several tabs).
2. Persist history in SQLite or Postgres (reusing Level 2 patterns) so it survives
   restarts; load the last 50 on join.
3. Implement the Redis pub/sub fan-out described above, run two instances on different
   ports, and prove a message sent to one reaches a client on the other.
4. Add a test for the heartbeat: create the server with `heartbeatMs: 50`, connect a raw
   TCP client that completes the upgrade but never answers pings, and assert the server
   terminates it.
