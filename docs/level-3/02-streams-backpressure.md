---
description: "Node streams and backpressure: readable, writable, transform and duplex streams, highWaterMark, write() returning false, drain, and stream.pipeline."
---

# 02 · Streams & Backpressure

A stream is data you process **piece by piece** instead of all at once. HTTP requests and
responses, files, sockets, child-process stdio, gzip, and crypto hashing are all streams
in Node. Streams let a 20 GB file flow through a process with a few megabytes of memory,
and let a response start before the whole result exists.

The hard part of streaming is not reading chunks — it's **backpressure**: what happens
when data arrives faster than the next stage can handle it.

## The four kinds

| Type | You… | Examples |
|------|------|----------|
| Readable | read from it | `fs.createReadStream`, `req` on a server, `process.stdin` |
| Writable | write to it | `fs.createWriteStream`, `res` on a server, `process.stdout` |
| Duplex | both, independent sides | TCP `net.Socket`, WebSocket streams |
| Transform | both, output derived from input | `zlib.createGzip()`, `crypto.createHash()`, a CSV parser |

## Backpressure, observed

A writable stream has an internal buffer. `write()` adds to it and returns **`true` if
the buffer is still below `highWaterMark`**, or **`false` if it's full** — meaning
"please stop until I emit `'drain'`". Nothing *forces* you to stop; if you ignore
`false`, data piles up in memory.

```js title="backpressure.mjs"
import { Writable } from 'node:stream';

// A deliberately slow destination: takes 10 ms per chunk
const slow = new Writable({
  highWaterMark: 4,          // bytes buffered before write() returns false
  write(chunk, encoding, callback) {
    setTimeout(callback, 10);
  },
});

let i = 0;
function writeMore() {
  while (i < 10) {
    const ok = slow.write(`${i}\n`);        // each chunk is 2 bytes
    console.log(`write ${i} -> ${ok}  (buffered: ${slow.writableLength} bytes)`);
    i++;
    if (!ok) {
      console.log('  full: waiting for drain');
      slow.once('drain', writeMore);
      return;
    }
  }
  slow.end(() => console.log('all written'));
}
writeMore();
```

```text
write 0 -> true  (buffered: 2 bytes)
write 1 -> false  (buffered: 4 bytes)
  full: waiting for drain
write 2 -> true  (buffered: 2 bytes)
write 3 -> false  (buffered: 4 bytes)
  full: waiting for drain
...
write 9 -> false  (buffered: 4 bytes)
  full: waiting for drain
all written
```

The producer writes until `false`, pauses, and resumes on `'drain'`. Memory is bounded
by `highWaterMark` regardless of how slow the consumer is. You'll rarely write this loop
by hand — `pipeline` does it for you — but this is exactly what it does.

## stream.pipeline: the right way to connect streams

`pipeline` connects stages, propagates backpressure between every pair, **destroys all
stages if any one fails**, and tells you when everything finished. Stages can be streams
or async generator functions, which are the easiest way to write a transform.

```js title="csv2ndjson.mjs"
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

// Async generator transform: bytes in, NDJSON lines out
async function* csvToNdjson(source) {
  let leftover = '';
  let header = null;
  for await (const chunk of source) {
    const lines = (leftover + chunk).split('\n');
    leftover = lines.pop();                    // last piece may be an incomplete line
    let out = '';
    for (const line of lines) {
      if (!line) continue;
      const cells = line.split(',');
      if (!header) { header = cells; continue; }
      const row = Object.fromEntries(header.map((h, i) => [h, cells[i]]));
      row.amount = Number(row.amount);
      out += JSON.stringify(row) + '\n';
    }
    if (out) yield out;                        // one output chunk per input chunk
  }
  if (leftover) throw new Error('file did not end with a newline');
}

const t0 = performance.now();
await pipeline(
  createReadStream('orders.csv', { encoding: 'utf8' }),
  csvToNdjson,
  createGzip(),
  createWriteStream('orders.ndjson.gz'),
);
const mb = (n) => (n / 1024 / 1024).toFixed(1);
console.log(`done in ${Math.round(performance.now() - t0)} ms, peak RSS ~${mb(process.resourceUsage().maxRSS * 1024)} MB`);
```

## Worked example: streaming vs loading everything

On a generated 20 MB CSV with one million rows, the streaming version above and a
"naive" version (`readFile` → `split('\n')` → `map` → `join` → `gzipSync` →
`writeFile`) produced byte-identical output. Observed on one laptop:

| Version | Time | Peak RSS |
|---------|------|----------|
| Streaming pipeline | 990 ms | ~91 MB |
| Load everything | 1170 ms | ~469 MB |

The streaming version's memory is dominated by Node's own baseline and stays roughly flat
as the file grows; the naive version's memory grows with the input (the whole string, an
array of a million strings, a million objects, the output string, and the compressed
buffer all exist at once). At 2 GB of input the naive version simply crashes.

One detail mattered a lot. The first version of the generator yielded **one tiny string
per row**. It produced the same output but took about **7 seconds**, because every
`yield` is a separate chunk that passes through the stream machinery and gzip
individually. Batching output to one chunk per input chunk cut that to under a second.
Streams have per-chunk overhead; keep chunks reasonably sized (kilobytes, not bytes).

## Streams in HTTP

`req` is readable and `res` is writable, so you can stream in both directions:

```js
import { createReadStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';

app.get('/export.csv', async (req, res) => {
  res.setHeader('content-type', 'text/csv');
  await pipeline(createReadStream('/data/export.csv'), res);
});
```

If the client is on a slow connection, the socket's buffer fills, `res.write` returns
`false`, `pipeline` pauses the file read, and your server holds only a small buffer per
slow client. If the client disconnects, `res` is destroyed, `pipeline` rejects with
`ERR_STREAM_PREMATURE_CLOSE`, and the file stream is closed — no leaked file descriptor.
(Express 5 routes that rejection to the error handler; since headers are already sent,
the error handler should just log it.)

## How It Actually Works

**Readable streams** have two modes. In *paused* mode, data sits in an internal buffer
until someone calls `read()`. In *flowing* mode (after attaching a `'data'` listener,
calling `pipe`, or iterating with `for await`), the stream pushes chunks out as they
arrive. Internally, a readable calls its `_read()` method to ask the source for more
whenever its buffer is below `highWaterMark`, and stops asking when the buffer is full.
For `fs.createReadStream`, `_read()` issues a thread-pool `read()` of up to 64 KiB.

**Writable streams** keep a queue of pending writes. `_write(chunk, enc, callback)` is
called for one chunk at a time; the next chunk waits until the callback is called.
`write()` returns `writableLength < highWaterMark`. When the queue empties after having
been full, the stream emits `'drain'`.

**Backpressure chains** through `pipe`/`pipeline`: when the destination's `write()`
returns `false`, the source is paused (`readable.pause()`), so it stops calling `_read`;
its buffer fills up; the OS-level source (a file read or a TCP socket) stops being read.
For a **TCP socket**, not reading means the kernel's receive buffer fills, the TCP window
advertised to the sender shrinks to zero, and the *remote machine* stops sending.
Backpressure therefore propagates across the network all the way to the original
producer — as long as no stage in between buffers without limit.

**Async iteration** (`for await (const chunk of readable)`) implements the same thing:
the stream only reads ahead up to `highWaterMark` while your loop body is busy.

## Common mistakes

- **Ignoring `write()`'s return value** in a loop → unbounded memory growth.
- **Using `.pipe()` without error handling.** `a.pipe(b)` does not forward errors or
  destroy `a` if `b` fails, leaking file descriptors. Use `pipeline`.
- **Mixing `'data'` listeners with `for await`** or `pipe` on the same stream.
- **Tiny chunks** (one per line or per byte) → large overhead.
- **Collecting a whole stream into memory** "just to be safe" (`Buffer.concat` of
  everything) on untrusted input without a size limit.
- **Forgetting `objectMode`** when a custom stream should carry objects rather than
  bytes/strings.

## Exercise

1. Remove the `if (!ok)` branch from `backpressure.mjs`, write 100,000 chunks, and print
   `slow.writableLength` after the loop. Explain the number.
2. Add a stage to `csv2ndjson.mjs` that drops rows with `amount < 10` and counts how many
   were dropped. Make sure the count is printed only after the pipeline succeeds.
3. Build `GET /orders.ndjson` that streams rows from a generator (simulate the DB with an
   async generator yielding 100 rows per 50 ms). Use `curl --limit-rate 1k` to confirm
   the generator slows down when the client is slow.
4. Compute a SHA-256 of a large file using `crypto.createHash` in a pipeline and compare
   with `shasum -a 256` (or `sha256sum`).
