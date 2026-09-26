---
description: "Buffers and binary data in Node: bytes vs characters, encodings, shared-memory slices, safe UTF-8 decoding, and a length-prefixed wire protocol."
---

# 03 · Buffers & Binary Data

Networks and disks deal in **bytes**, not JavaScript strings. Node's `Buffer` is the type
for raw bytes: every chunk from a TCP socket, every file read without an encoding, every
hash digest is a `Buffer`. You need to understand them to handle uploads, write
protocol parsers, work with crypto, and avoid subtle text-corruption bugs.

`Buffer` is a subclass of the standard `Uint8Array`, so everything that works on typed
arrays works on buffers, plus Node-specific helpers for encodings and numeric reads.

## Bytes are not characters

```js title="buffers.mjs"
const b = Buffer.from('héllo €', 'utf8');
console.log(b);
console.log('bytes:', b.length, 'chars:', 'héllo €'.length);
console.log(b.toString('hex'), '|', b.toString('base64'), '|', b.toString('base64url'));

const view = b.subarray(0, 5);     // shares memory with b
view[0] = 0x48;                    // 'H'
console.log(b.toString());

const copy = Buffer.from(b);       // independent copy
copy[0] = 0x4a;
console.log(b.toString(), copy.toString());

// A multi-byte character split across two chunks
const euro = Buffer.from('€');     // e2 82 ac
const [c1, c2] = [euro.subarray(0, 2), euro.subarray(2)];
console.log(JSON.stringify(c1.toString() + c2.toString()));
const { StringDecoder } = await import('node:string_decoder');
const dec = new StringDecoder('utf8');
console.log(JSON.stringify(dec.write(c1) + dec.write(c2)));
```

```text
<Buffer 68 c3 a9 6c 6c 6f 20 e2 82 ac>
bytes: 10 chars: 7
68c3a96c6c6f20e282ac | aMOpbGxvIOKCrA== | aMOpbGxvIOKCrA
Héllo €
Héllo € Jéllo €
"��"
"€"
```

Line by line:

1. In UTF-8, `h` is one byte, `é` is two (`c3 a9`), and `€` is three (`e2 82 ac`). The
   string has 7 characters but 10 bytes. **Use `Buffer.byteLength(str)` for
   `Content-Length`**, never `str.length`.
2. The same bytes can be shown as hex, base64, or base64url (URL-safe, no padding —
   used in JWTs).
3. **`subarray` does not copy.** Modifying `view` changed `b`. This is fast (no allocation)
   but means a small slice keeps the whole underlying memory alive.
4. `Buffer.from(buffer)` copies.
5. **Decoding split multi-byte characters.** If a chunk boundary falls inside `€`,
   calling `toString()` on each chunk yields two replacement characters (`��`).
   `StringDecoder` holds back incomplete sequences until the next chunk. This is exactly
   what `setEncoding('utf8')` on a stream does internally — which is why streams should
   be given an encoding rather than having each chunk `toString()`ed.

## Creating buffers

```js
Buffer.from('text', 'utf8');        // from a string
Buffer.from([0x01, 0xff]);          // from byte values
Buffer.alloc(1024);                 // 1 KiB of zeros
Buffer.allocUnsafe(1024);           // 1 KiB, NOT zeroed — may contain old memory
Buffer.concat([a, b, c]);           // join (copies)
```

`allocUnsafe` is faster because it skips zero-filling, and it may contain leftover data
from previously freed buffers — possibly secrets. Only use it when you will overwrite
every byte immediately, as `encodeFrame` below does.

## Reading and writing numbers

Binary formats store integers and floats in fixed widths and a specific **byte order**
(big-endian is "network byte order"):

```js
const header = Buffer.alloc(8);
header.writeUInt16BE(0xcafe, 0);   // bytes 0-1
header.writeUInt8(3, 2);           // byte 2: version
header.writeInt32LE(-42, 4);       // bytes 4-7, little-endian
header.readUInt16BE(0);            // 51966
header.readInt32LE(4);             // -42
```

Methods exist for 8/16/32-bit ints, `BigInt64`, floats, and doubles, in BE/LE forms.

## Worked example: a length-prefixed message protocol

TCP is a **byte stream**, not a message stream. If a client sends two messages, the
server might receive them in one chunk, or split across four chunks at arbitrary byte
positions. A common fix is **length-prefix framing**: each message is a 4-byte length
followed by that many bytes of payload.

```js title="framing.mjs"
// Length-prefixed framing: [4-byte big-endian length][payload]
export function encodeFrame(obj) {
  const payload = Buffer.from(JSON.stringify(obj), 'utf8');
  const frame = Buffer.allocUnsafe(4 + payload.length);
  frame.writeUInt32BE(payload.length, 0);
  payload.copy(frame, 4);
  return frame;
}

export class FrameDecoder {
  #buf = Buffer.alloc(0);
  push(chunk) {
    this.#buf = Buffer.concat([this.#buf, chunk]);
    const messages = [];
    while (this.#buf.length >= 4) {
      const len = this.#buf.readUInt32BE(0);
      if (len > 1_000_000) throw new Error(`frame too large: ${len}`);
      if (this.#buf.length < 4 + len) break;            // wait for the rest
      messages.push(JSON.parse(this.#buf.toString('utf8', 4, 4 + len)));
      this.#buf = this.#buf.subarray(4 + len);
    }
    return messages;
  }
}

// Simulate TCP delivering two frames split at arbitrary points
const wire = Buffer.concat([encodeFrame({ type: 'hello', user: 'ada' }), encodeFrame({ type: 'msg', text: 'hi €' })]);
console.log('wire bytes:', wire.length, wire.subarray(0, 8));
const d = new FrameDecoder();
for (const [a, b] of [[0, 3], [3, 20], [20, 40], [40, wire.length]]) {
  console.log(`chunk ${a}-${b}:`, d.push(wire.subarray(a, b)));
}
```

```text
wire bytes: 67 <Buffer 00 00 00 1d 7b 22 74 79>
chunk 0-3: []
chunk 3-20: []
chunk 20-40: [ { type: 'hello', user: 'ada' } ]
chunk 40-67: [ { type: 'msg', text: 'hi €' } ]
```

The first 4 bytes `00 00 00 1d` say "29 bytes follow". The decoder accumulates bytes,
emits a message only when a complete frame is present, and keeps the remainder. It also
**caps the frame size**; without that, a malicious length of 4 GB would make you buffer
forever. Plug `FrameDecoder` into a `net` server with
`socket.on('data', c => decoder.push(c).forEach(handle))` and you have a working
protocol. (Production code avoids re-`concat`ing on every chunk by keeping a list of
chunks, but the logic is the same.)

## How It Actually Works

A `Buffer` is a `Uint8Array` view over an `ArrayBuffer`, whose memory lives **outside the
V8 JavaScript heap**. That matters for memory accounting: `process.memoryUsage()` reports
buffer memory under `arrayBuffers`/`external`, not `heapUsed`, so a leak of buffers won't
show up as heap growth.

For small allocations, Node uses a **pool**: `Buffer.allocUnsafe(n)` and
`Buffer.from(string)` for sizes under half of `Buffer.poolSize` carve a slice out of a
shared pre-allocated `ArrayBuffer` instead of allocating a new one each time. (On the
Node version used here `Buffer.poolSize` printed `65536`; older releases used 8 KiB.)
It's fast, and it's one reason `allocUnsafe` can contain stale data: the pool is reused.
It also means `buf.buffer` may be much larger than `buf` — always use `buf.byteOffset`
and `buf.length` when handing `buf.buffer` to other APIs.

When a socket receives data, libuv reads from the kernel into memory Node allocated, and
Node wraps that memory as a Buffer without copying. `subarray` creates a new view with a
different offset and length over the same memory — which is why it's O(1).

## Common mistakes

- **`str.length` for byte counts** (Content-Length, size limits, DB column limits in bytes).
- **`chunk.toString()` per chunk** on multi-byte text → corrupted characters at chunk
  boundaries. Use `setEncoding`, `StringDecoder`, or collect and decode once.
- **Assuming one `'data'` event = one message** on TCP.
- **Keeping tiny slices of huge buffers** alive, retaining the whole allocation.
  `Buffer.from(slice)` copies it out if you need to keep it.
- **`allocUnsafe` without overwriting** every byte.
- **`new Buffer(...)`** — deprecated and unsafe; use `Buffer.from`/`Buffer.alloc`.

## Exercise

1. Write `byteTruncate(str, maxBytes)` that truncates a UTF-8 string to at most `maxBytes`
   bytes without splitting a character. Test with emoji.
2. Build a TCP echo server with `node:net` that uses `FrameDecoder`, and a client that
   sends 1,000 frames in a tight loop. Verify all arrive intact.
3. Parse the header of a PNG file: check the 8-byte signature, then read the IHDR chunk's
   width and height (big-endian 32-bit integers at offsets 16 and 20).
4. Allocate 200 MB of buffers in a loop, log `process.memoryUsage()` before and after,
   and identify which field grew.
