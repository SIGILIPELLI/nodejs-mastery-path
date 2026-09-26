---
description: "What Node.js really is: V8 running your JavaScript, libuv handling I/O, the single-threaded event loop, and the hidden thread pool."
---

# 01 · What Node.js Is: V8 + libuv

Node.js is not a language and not a framework. It is a **runtime**: a program (the
`node` executable) that takes JavaScript source and runs it outside a browser, with a
set of built-in modules for talking to the operating system — files, sockets,
processes, timers, cryptography.

Two pieces do most of the work:

- **V8** — Google's JavaScript engine, the same one inside Chrome. It parses your code,
  compiles it to machine code, runs it, and garbage-collects memory. V8 knows nothing
  about files or networks; it only executes JavaScript.
- **libuv** — a C library that provides an **event loop** and asynchronous access to
  the OS: non-blocking sockets, timers, file system operations, DNS, child processes,
  signals. It hides the differences between Linux (epoll), macOS (kqueue), and Windows
  (IOCP).

Node glues these together with C++ bindings and a JavaScript standard library
(`node:fs`, `node:http`, `node:stream`, and so on). When you call `fs.readFile`, your
JavaScript runs in V8, crosses into C++, asks libuv to do the read, and later libuv
tells the event loop "done" so your callback runs back in V8.

You can see the versions of each piece in your own install:

```bash
node -p "process.versions"
```

The output lists `node`, `v8`, `uv`, `openssl`, `zlib`, and more. These are the
components your `node` binary was built with.

## The one-thread rule

Your JavaScript runs on **one main thread**. There is no way for two of your functions
to execute at the same instant inside one Node process (unless you explicitly create
worker threads, covered in Level 3). This has two consequences that shape everything
else in this course.

**Good consequence:** no locks, no data races on your objects. While a function runs,
nothing else can modify the variables it is looking at.

**Bad consequence:** if a function takes a long time, *everything* waits. Run this:

```js title="block.js"
const t0 = Date.now();
setTimeout(() => console.log(`timer fired after ${Date.now() - t0} ms (asked for 10)`), 10);

// Busy loop: nothing else can run on this thread until it finishes
const end = Date.now() + 500;
while (Date.now() < end) {}
console.log('busy loop finished');
```

```text
busy loop finished
timer fired after 504 ms (asked for 10)
```

The timer was due after 10 ms, but the thread was busy. The event loop only gets a
chance to check timers once the current JavaScript finishes. In a web server, "the
timer" is every other user's request.

So why is Node good at servers? Because most server work is **waiting** — for a
database, a disk, another API. Node doesn't spend a thread per waiting request; it
registers interest with the OS and goes back to running JavaScript for whoever is
ready. Thousands of idle connections cost little more than their memory.

## Where the other threads are

"Node is single-threaded" is only true of your JavaScript. A running Node process has
several threads: V8's garbage-collector and compiler helpers, and libuv's **thread
pool** (4 threads by default). Some operations cannot be done asynchronously by the OS
in a portable way, so libuv runs them on the pool:

- most `fs` operations (file reads, writes, `stat`)
- `dns.lookup()` (which calls the system's `getaddrinfo`)
- CPU-heavy crypto such as `crypto.pbkdf2`, `crypto.scrypt`, `crypto.randomBytes`
  (async form)
- `zlib` compression in its async form

Network sockets do **not** use the pool; they use the OS's readiness notification
(epoll/kqueue/IOCP) directly on the event-loop thread.

You can watch the pool's size limit in action:

```js title="pool.js"
import { pbkdf2 } from 'node:crypto';

const start = performance.now();
for (let i = 1; i <= 6; i++) {
  pbkdf2('secret', 'salt', 200_000, 64, 'sha512', () => {
    console.log(`hash ${i} done at ${Math.round(performance.now() - start)} ms`);
  });
}
```

On one run on a laptop this printed (your timings will differ):

```text
hash 3 done at 54 ms
hash 4 done at 56 ms
hash 1 done at 57 ms
hash 2 done at 57 ms
hash 5 done at 105 ms
hash 6 done at 106 ms
```

Four hashes finish together, then the remaining two take a second "round". Only four
pool threads exist, so hashes 5 and 6 queue behind the first four. Run it with
`UV_THREADPOOL_SIZE=6 node pool.js` and all six finish at roughly the same time. The
file uses `import`, so name it `pool.mjs` or put `"type": "module"` in a
`package.json` next to it (lesson 03 explains why).

## Worked example: what happens when you run `node server.js`

```js title="server.js"
import { createServer } from 'node:http';
import { readFile } from 'node:fs/promises';

const server = createServer(async (req, res) => {
  const body = await readFile(new URL('./hello.txt', import.meta.url));
  res.writeHead(200, { 'content-type': 'text/plain' });
  res.end(body);
});

server.listen(3000, () => console.log('listening on http://localhost:3000'));
```

Step by step:

1. `node` starts, initializes V8 and libuv, and loads `server.js` as an ES module.
2. Your top-level code runs: it creates a server and calls `listen`. libuv opens a
   listening socket and registers it with the OS poller. Top-level code ends.
3. The event loop starts. Because there is an active handle (the listening socket),
   the process stays alive instead of exiting.
4. A request arrives. The OS marks the socket readable; libuv's poll phase notices,
   Node's HTTP parser (llhttp) parses the bytes, and your handler is called.
5. `readFile` hands the read to the thread pool and returns a pending promise. Your
   handler pauses at `await`. The main thread is free to serve other requests.
6. A pool thread finishes the read, libuv queues the completion, and the loop resumes
   your function with the file contents. You write the response; libuv sends it.

Create `hello.txt`, run `node server.js`, and `curl localhost:3000`.

## How It Actually Works

The event loop is a `while` loop in C inside libuv (`uv_run`). Each iteration
("tick") walks through phases in a fixed order:

1. **timers** — run callbacks whose `setTimeout`/`setInterval` time has passed
2. **pending callbacks** — some deferred system-level callbacks (e.g. certain TCP errors)
3. **idle/prepare** — internal housekeeping
4. **poll** — ask the OS "which of my sockets/fds have events?" and run their callbacks;
   this is where the loop *blocks* waiting when there is nothing else to do
5. **check** — run `setImmediate` callbacks
6. **close callbacks** — e.g. `socket.on('close')`

Between every callback, Node drains two extra queues: `process.nextTick` callbacks and
then promise microtasks. Level 3 spends a whole lesson on this ordering.

The loop keeps going while there are **active handles or requests**: a listening
server, an open socket, a pending timer, an in-flight file read. When none remain,
`uv_run` returns and the process exits. That is why a script with only
`console.log('hi')` exits immediately, while a server keeps running. You can let a
timer not keep the process alive with `timer.unref()`.

The thread pool is a fixed set of worker threads with a shared work queue. When you
call `fs.readFile`, Node submits a work item; a pool thread performs the blocking
`read()` system call; when it finishes, it signals the loop (via an internal async
handle), and in the next poll phase the completion callback runs on the main thread.
Your JavaScript never runs on pool threads — only the C-level blocking work does.

## Common mistakes

- **Thinking `async` makes CPU work non-blocking.** An `async function` that computes
  a large sum still runs on the main thread. `async` only helps when the function
  *waits* on something.
- **Using `*Sync` APIs in request handlers.** `fs.readFileSync` blocks the loop for the
  duration of the read. Fine in a startup script or CLI; bad inside a server handler.
- **Assuming "single-threaded" means only one thing at a time.** Many file reads and
  DNS lookups proceed in parallel on the pool; many socket operations proceed in
  parallel in the kernel.
- **Ignoring thread-pool saturation.** Heavy `pbkdf2`/`scrypt` hashing or many slow
  file reads can queue behind each other and delay unrelated `fs` calls and
  `dns.lookup`, even though the main thread is idle.

## Exercise

1. Run `node -p "process.versions"` and note your `v8` and `uv` versions.
2. Modify `pool.js` to start 8 hashes. Predict the timing pattern with the default pool,
   then with `UV_THREADPOOL_SIZE=2` and `=8`. Run all three and compare.
3. Add a `setInterval` that logs every 100 ms to `block.js`, then make the busy loop last
   one second. How many interval ticks do you lose? Explain what a user of a web server
   would experience in the same situation.
