---
description: "Handle file uploads safely in Node: multipart parsing with multer, size limits, content sniffing, generated filenames, and direct-to-storage uploads."
---

# 08 · File Uploads

File uploads look simple — "accept a file, save it" — and are one of the most common
sources of security bugs and outages in web backends. The file can be enormous, lie about
its type, carry a filename like `../../app/server.js`, or be an HTML page that runs
scripts when served back from your domain. This lesson builds an avatar upload endpoint
that handles all of that, then looks at the architecture most production systems end up
with: uploading directly to object storage.

## How browsers send files

An HTML form with `enctype="multipart/form-data"` (or a `FormData` object in `fetch`)
sends a **multipart** body: several parts separated by a random boundary string, each
with its own headers:

```text
POST /avatars HTTP/1.1
Content-Type: multipart/form-data; boundary=----x1234

------x1234
Content-Disposition: form-data; name="note"

profile picture
------x1234
Content-Disposition: form-data; name="avatar"; filename="me.png"
Content-Type: image/png

<binary bytes...>
------x1234--
```

`express.json()` doesn't parse this. You need a streaming multipart parser. **multer** is
the standard Express middleware; it's built on **busboy**, a fast streaming parser you can
also use directly. (Web-standard `request.formData()` exists in frameworks built on the
Fetch API, but buffers the whole body in memory.)

```bash
npm install multer
```

## Worked example: a safe avatar endpoint

```js title="app.js"
import express from 'express';
import multer from 'multer';
import { randomUUID } from 'node:crypto';
import { open, rm, mkdir } from 'node:fs/promises';
import path from 'node:path';

const UPLOAD_DIR = path.join(import.meta.dirname, 'uploads');
await mkdir(UPLOAD_DIR, { recursive: true });

const upload = multer({
  storage: multer.diskStorage({
    destination: UPLOAD_DIR,
    // Never use the client's filename on disk: generate our own
    filename: (req, file, cb) => cb(null, randomUUID()),
  }),
  limits: { fileSize: 2 * 1024 * 1024, files: 1, fields: 5 },
});

// Check the actual bytes, not the client-supplied mimetype or extension
const SIGNATURES = {
  'image/png': Buffer.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a]),
  'image/jpeg': Buffer.from([0xff, 0xd8, 0xff]),
};
async function sniffType(filePath) {
  const fh = await open(filePath, 'r');
  try {
    const { buffer } = await fh.read(Buffer.alloc(8), 0, 8, 0);
    return Object.entries(SIGNATURES).find(([, sig]) => buffer.subarray(0, sig.length).equals(sig))?.[0] ?? null;
  } finally {
    await fh.close();
  }
}

export const app = express();

app.post('/avatars', upload.single('avatar'), async (req, res) => {
  if (!req.file) return res.status(400).json({ error: 'avatar file is required' });
  const type = await sniffType(req.file.path);
  if (!type) {
    await rm(req.file.path, { force: true });
    return res.status(415).json({ error: 'only PNG or JPEG images are allowed' });
  }
  res.status(201).json({ id: req.file.filename, type, bytes: req.file.size, originalName: req.file.originalname });
});

app.use((err, req, res, next) => {
  if (err instanceof multer.MulterError) {
    const status = err.code === 'LIMIT_FILE_SIZE' ? 413 : 400;
    return res.status(status).json({ error: err.message, code: err.code });
  }
  next(err);
});
```

Exercising it with Supertest:

```js title="try-uploads.js"
import request from 'supertest';
import { app } from './app.js';

const png = Buffer.concat([Buffer.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a]), Buffer.alloc(100)]);
const fakePng = Buffer.from('<?php system($_GET["c"]); ?>');
const big = Buffer.concat([png, Buffer.alloc(3 * 1024 * 1024)]);

for (const [label, buf, name] of [['real png', png, 'me.png'], ['script renamed .png', fakePng, 'me.png'], ['3 MB file', big, 'big.png']]) {
  const r = await request(app).post('/avatars').attach('avatar', buf, { filename: name, contentType: 'image/png' });
  console.log(label.padEnd(20), r.status, JSON.stringify(r.body));
}
const r = await request(app).post('/avatars').field('note', 'no file');
console.log('no file'.padEnd(20), r.status, JSON.stringify(r.body));
```

```text
real png             201 {"id":"9f81c4f8-...","type":"image/png","bytes":108,"originalName":"me.png"}
script renamed .png  415 {"error":"only PNG or JPEG images are allowed"}
3 MB file            413 {"error":"File too large","code":"LIMIT_FILE_SIZE"}
no file              400 {"error":"avatar file is required"}
```

After these four requests, the uploads directory contained exactly one file: the rejected
script was deleted by our handler, and multer removed the partial 3 MB file itself when
the limit was hit.

The defenses, one by one:

1. **Limits** (`fileSize`, `files`, `fields`). multer stops reading as soon as the limit
   is crossed. Without a limit, a single request can fill your disk.
2. **Generated filenames.** The client's `originalname` is untrusted: it can contain
   path separators, `..`, or names that collide. Store under a random id; keep the
   original name only as metadata (and sanitize it before echoing it anywhere).
3. **Content sniffing.** The `Content-Type` header and the extension are chosen by the
   client — the second test sent a PHP script labelled `image/png`. Checking the file's
   **magic bytes** is much stronger. (Packages like `file-type` recognize many formats.)
   For images, re-encoding with an image library (e.g. `sharp`) is stronger still: it
   produces a clean file and strips metadata such as GPS coordinates.
4. **Errors mapped to status codes** — 413 for size, 415 for type, 400 for missing file.

## Serving uploaded files safely

If you serve user uploads from your main domain, a file containing HTML/JavaScript can run
in your origin (stored XSS). Mitigations:

- Serve from a **separate domain** (e.g. a storage bucket or `usercontent.example.com`).
- Set `Content-Type` from your sniffed type, plus `X-Content-Type-Options: nosniff`.
- For downloads, `Content-Disposition: attachment; filename="..."`.

## The production pattern: direct-to-storage uploads

Streaming every upload through your Node servers costs bandwidth, disk, and event-loop
time, and local disk doesn't work when you run several containers. Most production
systems store files in **object storage** (Amazon S3, Google Cloud Storage, Azure Blob,
Cloudflare R2, MinIO) and let the browser upload **directly** to it:

1. Client: `POST /uploads` with `{ contentType, size }`.
2. Server: validates the request and user quota, generates a key like
   `avatars/<userId>/<uuid>`, and returns a **pre-signed URL** — a URL containing a
   time-limited signature that authorizes exactly one upload to that key (for S3, via
   `@aws-sdk/s3-request-presigner`). Constraints on size and content type can be baked into
   the signed request.
3. Client: `PUT`s the file straight to storage.
4. Client: `POST /uploads/<id>/complete`; server verifies the object exists, and a
   background job (Level 4) sniffs/re-encodes/scans it before marking it usable.

This requires a cloud or self-hosted storage service, so it isn't runnable in this
lesson's local setup; the local version above teaches the same validation steps you apply
in step 4.

## How It Actually Works

multer registers itself as middleware that, for `multipart/form-data` requests, pipes
`req` (a readable stream) into busboy. busboy scans the incoming bytes for the boundary
string, parses each part's headers, and emits `'field'` events for text fields and
`'file'` events that provide a **readable stream per file**. multer's disk storage pipes
that stream into `fs.createWriteStream` — so the file flows from socket to disk in chunks
and never sits in memory whole. Backpressure applies end to end: if the disk is slow, the
write stream fills, busboy pauses, `req` pauses, and the TCP window closes on the client.

The `fileSize` limit is enforced by counting bytes as they stream through busboy; when
exceeded, the file stream is truncated, multer stops, deletes the partial file, and calls
`next` with a `MulterError('LIMIT_FILE_SIZE')`.

`memoryStorage()` instead collects each file into a `Buffer` (`req.file.buffer`) —
convenient for small files you'll process immediately, dangerous without tight limits.

## Common mistakes

- **No size limits** (or relying on a reverse proxy's limit you never configured).
- **Using `originalname` as the path** → path traversal and overwrites.
- **Trusting `mimetype`/extension.**
- **Serving uploads from the app's origin** with a sniffable content type.
- **`memoryStorage` for large files** → memory spikes and crashes under concurrency.
- **Leaving orphaned files** when validation fails after the upload. Clean up in every
  failure path.
- **Storing uploads on a container's local disk** — lost on redeploy, invisible to other
  replicas.

## Exercise

1. Extend `sniffType` to recognize GIF (`GIF87a`/`GIF89a`) and WebP (`RIFF....WEBP`), with
   tests.
2. Add `GET /avatars/:id` that validates the id as a UUID, streams the file with the
   sniffed content type, and sets `X-Content-Type-Options: nosniff`.
3. Replace disk storage with a custom multer storage engine (`_handleFile`,
   `_removeFile`) that computes a SHA-256 of the file while writing it and rejects
   duplicates.
4. Sketch (in code, without running it against a cloud) the `POST /uploads` and
   `/complete` endpoints for the pre-signed flow, including what you store in the database
   at each step.
