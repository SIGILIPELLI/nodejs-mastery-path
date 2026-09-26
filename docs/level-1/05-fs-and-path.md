---
description: "Work with files and paths in Node: fs/promises, sync vs async vs streaming reads, atomic writes, directory walking, and safe path handling."
---

# 05 · Files & Paths: fs and path

Almost every backend touches the file system: config files, uploads, logs, generated
reports, caches. Node's `node:fs` module gives you three API styles for the same
operations, and `node:path` keeps you from building paths with string concatenation.

## Three flavors of fs

```js
import { readFile } from 'node:fs/promises';   // promise-based (use this by default)
import { readFileSync } from 'node:fs';         // synchronous, blocks the thread
import { readFile as readFileCb } from 'node:fs'; // callback-based (older style)
```

| Style | When to use |
|-------|------------|
| `fs/promises` + `await` | Default for application code and servers |
| `*Sync` | Startup code, CLIs, build scripts — where blocking is harmless |
| Callbacks | Older code; no advantage in new code |
| Streams (`createReadStream`) | Large files, or when you want to start processing before the whole file is in memory |

## Everyday operations

```js
import { readFile, writeFile, appendFile, mkdir, readdir, stat, rm, rename, access }
  from 'node:fs/promises';
import { constants } from 'node:fs';

const text = await readFile('notes.txt', 'utf8');          // string (omit 'utf8' → Buffer)
await writeFile('out.json', JSON.stringify({ ok: true }, null, 2));
await appendFile('app.log', `${new Date().toISOString()} started\n`);
await mkdir('data/cache', { recursive: true });              // like mkdir -p; no error if exists
const names = await readdir('data');                          // ['cache', ...]
const info = await stat('notes.txt');                         // size, mtime, isFile(), ...
await rename('out.json', 'data/out.json');                    // move (same filesystem)
await rm('data/cache', { recursive: true, force: true });     // like rm -rf — careful

try {
  await access('config.json', constants.R_OK);
} catch {
  console.log('config.json missing or unreadable');
}
```

Prefer attempting the operation and handling the error over checking first with
`access`/`exists`. Between the check and the use, another process can delete the file
(a "time-of-check to time-of-use" race). Handle errors by `code`:

```js
try {
  const raw = await readFile('settings.json', 'utf8');
} catch (err) {
  if (err.code === 'ENOENT') { /* use defaults */ }
  else throw err;   // EACCES, EISDIR, ... are real problems
}
```

## The path module

```js title="paths.mjs"
import path from 'node:path';

console.log(path.join('/srv/app', 'uploads', '../logs', 'app.log'));
console.log(path.parse('/srv/app/logs/app.2024-01-01.log'));
console.log(path.extname('archive.tar.gz'), path.basename('/a/b/report.pdf', '.pdf'));
console.log(path.relative('/srv/app/src', '/srv/app/public/index.html'));
```

```text
/srv/app/logs/app.log
{
  root: '/',
  dir: '/srv/app/logs',
  base: 'app.2024-01-01.log',
  ext: '.log',
  name: 'app.2024-01-01'
}
.gz report
../public/index.html
```

- `path.join` concatenates and normalizes (resolves `..`, removes duplicate slashes).
- `path.resolve` produces an **absolute** path, resolving relative segments against
  `process.cwd()` — the directory you ran `node` from, *not* the script's directory.
- Use `path.posix` / `path.win32` explicitly if you must produce a specific style.

To refer to a file next to your module regardless of where `node` was launched from,
use `import.meta.dirname`:

```js
const templatePath = path.join(import.meta.dirname, 'templates', 'email.html');
```

## Worked example: a disk-usage report

This walks a directory tree and totals file sizes by extension:

```js title="du.mjs"
import { readdir, stat } from 'node:fs/promises';
import path from 'node:path';

async function du(dir) {
  const byExt = new Map();
  let total = 0;
  const entries = await readdir(dir, { recursive: true, withFileTypes: true });
  for (const entry of entries) {
    if (!entry.isFile()) continue;
    const full = path.join(entry.parentPath, entry.name);
    const { size } = await stat(full);
    total += size;
    const ext = path.extname(entry.name) || '(none)';
    byExt.set(ext, (byExt.get(ext) ?? 0) + size);
  }
  return { total, byExt };
}

const target = process.argv[2] ?? '.';
const { total, byExt } = await du(target);
console.log(`${target}: ${total} bytes`);
for (const [ext, bytes] of [...byExt].sort((a, b) => b[1] - a[1])) {
  console.log(`  ${ext.padEnd(8)} ${bytes}`);
}
```

On a small test folder containing `one.txt` ("hello\n"), `three.txt` ("x\n"), and an
8-byte `two.log` in a subfolder, `node du.mjs demo` printed:

```text
demo: 16 bytes
  .txt     8
  .log     8
```

`readdir` with `recursive: true` and `withFileTypes: true` returns `Dirent` objects
whose `parentPath` tells you which subdirectory each entry came from.

## Writing files safely

`writeFile` truncates the file and then writes. If the process crashes halfway, you
are left with a half-written file. For files that must never be corrupt (config,
state), write to a temporary file and **rename** it into place:

```js
import { writeFile, rename } from 'node:fs/promises';

export async function writeJsonAtomic(file, data) {
  const tmp = `${file}.${process.pid}.tmp`;
  await writeFile(tmp, JSON.stringify(data, null, 2));
  await rename(tmp, file);   // atomic replace on POSIX filesystems
}
```

## How It Actually Works

Every `fs/promises` call becomes a request to libuv, which runs the blocking system call
(`open`, `read`, `fstat`, `rename`...) on a **thread-pool** thread and then resolves your
promise back on the main thread. Unlike sockets, regular files are always "ready" as far
as epoll/kqueue are concerned, so there is no portable non-blocking file I/O. That is why
fs uses the pool — and why a flood of slow file reads (e.g. on a network drive) can
starve other pool users like `dns.lookup` or `crypto.pbkdf2`.

`readFile` first `fstat`s the file to learn its size, allocates a buffer, reads in
chunks until done, and closes the descriptor. The whole file ends up in memory — fine
for kilobytes, dangerous for gigabytes. `createReadStream` instead reads fixed-size
chunks (64 KiB by default) on demand, so memory stays flat (Level 3, lesson 02).

`rename` is atomic because on POSIX systems it updates a directory entry in a single
operation: readers see either the old file or the new one, never a mixture. It only
works within one filesystem; across mount points it fails with `EXDEV`.

Paths are just strings until they reach the kernel. `path.join` does not touch the disk
and does not check that anything exists.

## Common mistakes

- **Path traversal.** `path.join(uploadsDir, req.params.name)` with a name of
  `../../etc/passwd` escapes your folder. Resolve the path and verify it still starts
  with the base directory plus a separator before using it.
- **Confusing `process.cwd()` with the script directory.** Running `node src/app.js`
  from the project root vs from `src/` changes what `'./data.json'` means.
- **Reading huge files with `readFile`.** Use streams.
- **Using `fs.exists` checks before operations.** Race-prone; handle `ENOENT` instead.
- **`rm -rf` with a computed path.** An empty or wrong variable can delete far more than
  intended. Validate the path first.

## Exercise

1. Extend `du.mjs` to skip `node_modules` and `.git`, and to print the five largest
   files with their sizes in KB.
2. Write `safeJoin(base, userPath)` that throws if the result escapes `base`. Test it
   with `"a.txt"`, `"../x"`, `"sub/../../x"`, and an absolute path like `"/etc/passwd"`.
3. Write a script that keeps a JSON counter file, increments it on each run using
   `writeJsonAtomic`, and survives being killed with Ctrl+C mid-run.
