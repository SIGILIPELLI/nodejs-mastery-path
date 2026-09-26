---
description: "Run CPU-heavy work off the event loop with worker_threads, and external programs with child_process: execFile, spawn, fork, IPC, and safe arguments."
---

# 04 · Worker Threads & Child Processes

The event loop is great at waiting and terrible at computing. Resizing an image, parsing
a 50 MB JSON document, hashing, compressing, or running a report calculation on the main
thread blocks every other request (Level 3, lesson 01). Node gives you two escape hatches:

- **`worker_threads`** — additional JavaScript threads *inside the same process*, each
  with its own V8 isolate and event loop. For CPU-bound JavaScript.
- **`child_process`** — separate OS processes. For running other programs (`ffmpeg`,
  `git`, a Python script) or for strong isolation.

## Worker threads, measured

A deliberately CPU-heavy function:

```js title="primes.js"
// CPU-bound: count primes below n (deliberately naive)
export function countPrimes(n) {
  let count = 0;
  for (let i = 2; i < n; i++) {
    let prime = true;
    for (let j = 2; j * j <= i; j++) if (i % j === 0) { prime = false; break; }
    if (prime) count++;
  }
  return count;
}
```

The worker file receives jobs by message and posts results back:

```js title="prime-worker.js"
import { parentPort } from 'node:worker_threads';
import { countPrimes } from './primes.js';

parentPort.on('message', ({ id, n }) => {
  parentPort.postMessage({ id, result: countPrimes(n) });
});
```

A comparison script with a 10 ms "heartbeat" interval on the main thread, to show
whether the main thread stays responsive:

```js title="worker-demo.js"
import { Worker } from 'node:worker_threads';
import { countPrimes } from './primes.js';

// Heartbeat: shows whether the main thread is free
let beats = 0;
const hb = setInterval(() => beats++, 10);

async function measure(label, fn) {
  beats = 0;
  const t0 = performance.now();
  const result = await fn();
  console.log(`${label}: result=${result} in ${Math.round(performance.now() - t0)} ms, heartbeats=${beats}`);
}

await measure('main thread ', async () => countPrimes(5_000_000));

const worker = new Worker(new URL('./prime-worker.js', import.meta.url));
await measure('worker      ', () => new Promise((resolve, reject) => {
  worker.once('message', (m) => resolve(m.result));
  worker.once('error', reject);
  worker.postMessage({ id: 1, n: 5_000_000 });
}));

// Four workers, four independent jobs, running at the same time
const workers = Array.from({ length: 4 }, () => new Worker(new URL('./prime-worker.js', import.meta.url)));
await measure('4 workers x4', async () => {
  const results = await Promise.all(workers.map((w, i) => new Promise((resolve) => {
    w.once('message', (m) => resolve(m.result));
    w.postMessage({ id: i, n: 5_000_000 });
  })));
  return results.join(',');
});

clearInterval(hb);
await worker.terminate();
await Promise.all(workers.map(w => w.terminate()));
```

Observed on an 8-core laptop:

```text
main thread : result=348513 in 852 ms, heartbeats=0
worker      : result=348513 in 873 ms, heartbeats=74
4 workers x4: result=348513,348513,348513,348513 in 945 ms, heartbeats=81
```

- On the main thread the computation took 852 ms and the heartbeat **never ran**: the
  process was frozen for everyone.
- In a worker it took about as long (workers don't make code faster), but the heartbeat
  kept ticking — the main thread was free to serve requests.
- Four workers did **four times the work in roughly the same wall time**, because they
  ran on four cores in parallel.

## Worker pools

Starting a worker costs tens of milliseconds and some memory (a whole V8 isolate), so
don't spawn one per request. Keep a **pool** of long-lived workers sized to your cores
(`os.availableParallelism()`) and queue jobs to them. The `piscina` package is a
well-established pool implementation:

```js
import Piscina from 'piscina';
const pool = new Piscina({ filename: new URL('./resize-worker.js', import.meta.url).href });
app.post('/thumbnails', async (req, res) => {
  const out = await pool.run({ path: req.file.path, width: 200 });
  res.json(out);
});
```

### Sharing data

`postMessage` **copies** data using the structured clone algorithm. For large binary
payloads, pass an `ArrayBuffer` in the transfer list to move it without copying (the
sender loses access): `worker.postMessage(buf, [buf.buffer])`. `SharedArrayBuffer` plus
`Atomics` allow true shared memory for specialized cases.

## Child processes

```js title="child.mjs"
import { execFile, spawn } from 'node:child_process';
import { promisify } from 'node:util';

const execFileP = promisify(execFile);

// 1. Run a command, collect its output (bounded by maxBuffer)
const { stdout } = await execFileP('node', ['-p', 'process.version'], { timeout: 5_000 });
console.log('child node version:', stdout.trim());

// 2. Stream a long-running process's output line by line
const child = spawn('node', ['-e', `
  for (let i = 1; i <= 3; i++) setTimeout(() => console.log('progress ' + i * 33 + '%'), i * 100);
  setTimeout(() => { console.error('warning: disk almost full'); process.exit(3); }, 400);
`]);
child.stdout.setEncoding('utf8').on('data', (d) => process.stdout.write(`[child] ${d}`));
child.stderr.setEncoding('utf8').on('data', (d) => process.stdout.write(`[child stderr] ${d}`));
const [code] = await new Promise((resolve, reject) => {
  child.once('error', reject);                // e.g. ENOENT: command not found
  child.once('close', (...args) => resolve(args));
});
console.log('child exited with code', code);

// 3. Arguments are passed as an array: no shell, so no injection
const filename = 'report; rm -rf ~';          // hostile input stays one harmless argument
const r = await execFileP('node', ['-p', 'process.argv[1]', filename]);
console.log('argument received literally:', r.stdout.trim());
```

```text
child node version: v26.3.0
[child] progress 33%
[child] progress 66%
[child] progress 99%
[child stderr] warning: disk almost full
child exited with code 3
argument received literally: report; rm -rf ~
```

Choosing the function:

| Function | Shell? | Output | Use for |
|----------|--------|--------|---------|
| `execFile(cmd, args)` | no | buffered (limit: `maxBuffer`) | short commands with small output |
| `spawn(cmd, args)` | no (unless `shell: true`) | streams | long-running or large output |
| `exec('cmd string')` | **yes** | buffered | only with fully trusted, constant strings |
| `fork(modulePath)` | no | streams + IPC channel | another Node process you control |

**Never pass user input to `exec` or `spawn(..., { shell: true })`.** A filename of
`report; rm -rf ~` becomes two shell commands. With `execFile`/`spawn` and an argument
array, as the last example shows, the hostile string arrives as one literal argument.

`fork()` starts a Node script with a built-in message channel
(`child.send(obj)` / `process.on('message')`), useful when you want process-level
isolation — a crash or memory leak in the child can't take down the parent.

## Workers vs child processes vs cluster

| Need | Tool |
|------|------|
| CPU-heavy JavaScript without blocking requests | worker threads (pool) |
| Run an external program | `spawn`/`execFile` |
| Isolate untrusted or crash-prone code | child process (`fork`) |
| Serve HTTP on all cores | cluster or multiple processes (next lesson) |

## How It Actually Works

**A worker thread** is an OS thread running a complete, separate Node environment: its
own V8 isolate (separate heap and garbage collector), its own libuv event loop, and its
own copy of the modules it imports. Nothing is shared by default — that's why there are
no data races on JavaScript objects. `postMessage` serializes the value with V8's
structured-clone serializer into bytes, puts them on a message port queue, and the
receiving thread's loop deserializes them into new objects. Transferring an
`ArrayBuffer` hands over ownership of the underlying memory instead of copying it.

Note that all workers in a process still share **one libuv thread pool** (the
`UV_THREADPOOL_SIZE` pool from Level 1) for fs and crypto calls.

**A child process** is created with the OS's process-creation calls (`posix_spawn` or
`fork`+`exec` on Unix, `CreateProcess` on Windows), which libuv wraps in `uv_spawn`.
The child's stdin/stdout/stderr are connected to **pipes**, which the parent sees as
streams — with backpressure: if you don't read a child's stdout, the pipe buffer (often
64 KiB on Linux) fills and the child blocks on its next write. With `execFile`, Node
reads everything into memory and kills the child if output exceeds `maxBuffer`.
`fork()` adds an extra pipe for IPC, over which messages are sent as serialized JSON (or
the `advanced` serialization mode, which uses structured clone).

## Common mistakes

- **One worker per request** — startup cost and memory explode under load. Pool them.
- **Moving I/O-bound work to workers** — pointless; the event loop already handles I/O
  concurrently.
- **Sending huge objects** through `postMessage` repeatedly; the cloning cost can exceed
  the computation. Transfer buffers, or pass file paths.
- **Shell injection** via `exec` or `shell: true` with user input.
- **Not reading child stdout/stderr** → the child hangs when the pipe fills.
- **Forgetting `'error'` on child processes** → an `ENOENT` (command not found) crashes
  the parent.
- **Orphaned children** when the parent exits; kill them in your shutdown handler.

## Exercise

1. Split `countPrimes(20_000_000)` into four ranges and run them on four workers in
   parallel. Compare total time with a single worker.
2. Build `POST /hash` that hashes the request body with `pbkdf2Sync` 200,000 iterations —
   first on the main thread, then in a worker pool of size 4. Use `autocannon` or a small
   script of concurrent `fetch` calls to compare `/health` latency under load.
3. Write a function `runGit(args, cwd)` using `execFile` with a timeout that returns
   `stdout` and throws an error with `stderr` included on non-zero exit.
4. Use `fork()` to run a child that computes something and reports progress messages;
   kill it with `child.kill('SIGTERM')` after two seconds and handle the `'exit'` event.
