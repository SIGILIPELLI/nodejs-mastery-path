---
description: "WebSockets in Node: the HTTP upgrade handshake, a ws server with broadcast and heartbeats, the built-in client, auth, and scaling across processes."
---

# 07 · WebSockets

HTTP is request/response: the client asks, the server answers. For chat, live
dashboards, multiplayer games, collaborative editing, or notifications, the **server**
needs to push data whenever something happens. **WebSockets** provide a long-lived,
full-duplex connection over which either side can send messages at any time.

Alternatives worth knowing:

- **Server-Sent Events (SSE)** — a plain HTTP response that stays open and streams
  `data:` lines. One-way (server → client), works through most proxies, auto-reconnects in
  browsers. Often enough for notifications and dashboards.
- **Long polling** — the client makes a request the server holds until there's news.
  A fallback, rarely the first choice today.
- **Socket.IO** — a library on top of WebSockets adding rooms, acknowledgements,
  reconnection, and fallbacks. It uses its own protocol, so its clients and servers only
  talk to each other.

This lesson uses the **`ws`** package on the server — small, fast, and widely used — and
the `WebSocket` client that current Node releases provide as a global, with the same API
as in browsers.

```bash
npm install ws
```

## A broadcast server

```js title="server.js"
import { createServer } from 'node:http';
import { WebSocketServer } from 'ws';

const server = createServer((req, res) => res.end('ok'));
const wss = new WebSocketServer({ server, path: '/ws', maxPayload: 64 * 1024 });

wss.on('connection', (socket, req) => {
  socket.isAlive = true;
  socket.on('pong', () => { socket.isAlive = true; });
  socket.on('error', (err) => console.error('socket error', err.message));

  socket.on('message', (data, isBinary) => {
    if (isBinary) return socket.close(1003, 'text only');
    let msg;
    try { msg = JSON.parse(data.toString()); } catch { return socket.close(1007, 'invalid JSON'); }
    // Broadcast to every open client (including the sender)
    const out = JSON.stringify({ from: req.socket.remotePort, text: String(msg.text).slice(0, 500) });
    for (const client of wss.clients) {
      if (client.readyState === client.OPEN) client.send(out);
    }
  });
  socket.send(JSON.stringify({ system: `welcome, ${wss.clients.size} connected` }));
});

// Heartbeat: terminate connections that stop answering pings (dead NAT paths, sleeping laptops)
const interval = setInterval(() => {
  for (const socket of wss.clients) {
    if (!socket.isAlive) { socket.terminate(); continue; }
    socket.isAlive = false;
    socket.ping();
  }
}, 30_000);
wss.on('close', () => clearInterval(interval));

server.listen(3300, () => console.log('ws://localhost:3300/ws'));
```

And a client script using the global `WebSocket`:

```js title="client.js"
// Uses the WebSocket client built into current Node releases (same API as browsers)
function connect(name) {
  return new Promise((resolve) => {
    const ws = new WebSocket('ws://localhost:3300/ws');
    ws.addEventListener('message', (e) => console.log(`${name} got:`, e.data));
    ws.addEventListener('open', () => resolve(ws));
  });
}
const alice = await connect('alice');
const bob = await connect('bob');
alice.send(JSON.stringify({ text: 'hi bob' }));
setTimeout(() => { alice.close(); bob.close(); }, 200);
```

```text
alice got: {"system":"welcome, 1 connected"}
bob got: {"system":"welcome, 2 connected"}
alice got: {"from":50660,"text":"hi bob"}
bob got: {"from":50660,"text":"hi bob"}
```

Details that matter in the server:

- **Shared HTTP server.** The WebSocket server attaches to a normal `http.Server`, so
  one port serves both your REST API and `/ws`. With Express, pass the server returned by
  `app.listen()`.
- **`maxPayload`** caps message size (the `ws` default is much larger); a client sending a
  100 MB frame is rejected instead of buffered.
- **Validate every message.** Treat incoming frames exactly like request bodies: parse
  defensively, check the schema (Zod works fine here), limit lengths. Close codes like
  `1003` (unsupported data) and `1007` (invalid payload) tell well-behaved clients why.
- **Heartbeats.** A TCP connection can die silently — a phone switches networks, a NAT
  entry times out — and neither side notices until it tries to send. Periodic pings with
  a pong check find and clean up dead sockets; otherwise your `clients` set leaks.
- **Check `readyState` before sending**, and add an `'error'` listener on each socket.

## Authentication

Browsers can't set custom headers (like `Authorization`) on WebSocket connections. Common
approaches:

1. **Cookies**: the upgrade request carries the site's cookies, so a session cookie
   (Level 2, lesson 07) authenticates it. Also check the `Origin` header to block other
   sites from opening sockets with your users' cookies (cross-site WebSocket hijacking).
2. **A short-lived token** in the query string (`/ws?token=...`) or in the first message.
   Tokens in URLs may appear in logs, so keep them single-use and short-lived.

Authenticate **during the upgrade**, before accepting the connection:

```js
const wss = new WebSocketServer({ noServer: true });
server.on('upgrade', async (req, socket, head) => {
  try {
    const user = await authenticateUpgrade(req);          // throws if invalid
    wss.handleUpgrade(req, socket, head, (ws) => wss.emit('connection', ws, req, user));
  } catch {
    socket.write('HTTP/1.1 401 Unauthorized\r\n\r\n');
    socket.destroy();
  }
});
```

## Backpressure on sockets

`ws.send()` queues data in memory if the network is slower than you send. For a slow
client receiving a busy feed, `socket.bufferedAmount` grows without limit. Check it and
drop messages or disconnect clients that fall too far behind:

```js
if (client.bufferedAmount > 1_000_000) client.terminate();   // > ~1 MB queued
else client.send(out);
```

## Scaling beyond one process

`wss.clients` only contains sockets connected to **this** process. With several
processes (lesson 05) or containers, a message received by process A must also reach
users connected to process B. The usual solution is a **pub/sub** backbone: each process
publishes incoming messages to a Redis channel and subscribes to it, relaying what it
receives to its local clients. The Level 3 project implements the local part and
describes this extension.

## How It Actually Works

A WebSocket connection starts as an HTTP/1.1 request:

```text
GET /ws HTTP/1.1
Host: localhost:3300
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
```

The server answers `101 Switching Protocols` with `Sec-WebSocket-Accept`, computed as
base64(SHA-1(key + a fixed GUID defined in RFC 6455)). That proves the server understood
the WebSocket request (it's not an anti-forgery measure). From then on, the same TCP
socket carries **frames** instead of HTTP.

In Node, the HTTP server emits an `'upgrade'` event instead of `'request'` for such
requests, handing over the raw `net.Socket`. `ws` validates the headers, writes the 101
response, and then parses frames from the socket's byte stream: each frame has a small
header with an opcode (text, binary, close, ping, pong, continuation), a FIN bit
(fragmentation), a payload length (7 bits, or extended to 16 or 64 bits), and — for frames
from clients — a 4-byte **mask** XORed over the payload. Masking prevents a malicious page
from crafting bytes that confuse intermediary caches. Pings and pongs are control frames
handled by the library.

Each connection is just an open socket in libuv's poller, so an idle WebSocket costs
memory (buffers, your per-connection state) but no CPU. A single Node process can hold a
very large number of mostly idle connections; the limits you hit first are usually file
descriptors (`ulimit -n`), memory, and the CPU cost of broadcasting to many clients.

## Common mistakes

- **No heartbeat** → dead connections accumulate.
- **Trusting message content** — no validation, no size limits.
- **Missing `Origin` check** with cookie-based auth.
- **Broadcasting with `JSON.stringify` inside the loop** — serialize once, send the same
  string to everyone.
- **Ignoring `bufferedAmount`** for slow consumers.
- **Assuming in-memory client lists are global** when running multiple processes.

## Exercise

1. Add rooms: messages `{ "type": "join", "room": "general" }` and
   `{ "type": "say", "room": "general", "text": "..." }`; only room members receive `say`.
2. Validate messages with a Zod discriminated union on `type`, closing with 1007 on
   invalid input.
3. Implement `/events` as Server-Sent Events that pushes the current time every second,
   and consume it with `curl -N`. Compare the code with the WebSocket version.
4. Write a load script that opens 2,000 WebSocket clients using the global `WebSocket`
   and measure the server's memory with `process.memoryUsage()` before and after.
