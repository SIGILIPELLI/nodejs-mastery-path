---
description: "Level 3 of Node.js: event loop internals, streams and backpressure, buffers, worker threads, clustering, Redis caching, WebSockets, uploads, and security."
---

# Level 3 · Advanced <span class="level-badge">Under the Hood</span>

Goal: understand *why* Node behaves the way it does under load, and use that knowledge to
build services that stay fast and safe. Level 2 gave you a working API; this level
explains what's happening beneath it — the exact order of the event loop, how data flows
through streams without exhausting memory, what a Buffer really is — and then uses those
mechanics to add capabilities: CPU work off the main thread, all cores in use, a shared
cache, real-time connections, and file uploads.

Many lessons here include measurements taken while writing them (timings, memory, event
loop delay). Your numbers will differ by machine; the *shape* of the results is what
matters, and re-running the experiments yourself is part of the lesson.

## Modules

1. [The Event Loop in Depth](01-event-loop-deep-dive.md) — phases, nextTick vs microtasks (and why ESM differs), starvation, measuring loop delay
2. [Streams & Backpressure](02-streams-backpressure.md) — `write()` returning false, `pipeline`, and streaming vs loading everything, measured
3. [Buffers & Binary Data](03-buffers.md) — bytes vs characters, shared slices, and a length-prefixed protocol parser
4. [Worker Threads & Child Processes](04-worker-threads-child-processes.md) — keeping CPU work off the loop, pools, and safely running other programs
5. [Clustering & Using Every Core](05-clustering.md) — `node:cluster`, how connections are handed to workers, and when containers replace it
6. [Caching with Redis](06-caching-redis.md) — cache-aside, invalidation, stampede protection, and key design
7. [WebSockets](07-websockets.md) — the upgrade handshake, a `ws` server, heartbeats, auth, and scaling out
8. [File Uploads](08-file-uploads.md) — multipart parsing, limits, content sniffing, and direct-to-storage uploads
9. [Security for Node Services](09-security.md) — OWASP risks in Node terms, ReDoS, prototype pollution, SSRF, and the supply chain
10. [Project — A Real-Time Chat App](10-project-realtime-app.md) — authenticated WebSocket rooms with history, rate limits, and tests

## What you need before starting

- Levels 1 and 2. Lessons refer back to the tasks API and its middleware.
- Comfort reading short C-like pseudocode: a few "How It Actually Works" sections sketch
  libuv internals.
- Optional: a local Redis server for lesson 06 and the project's scale-out exercise. Every
  runnable example in this level works without one; where Redis is required, the lesson
  says so explicitly.
