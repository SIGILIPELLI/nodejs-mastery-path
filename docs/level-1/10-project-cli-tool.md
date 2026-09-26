---
description: "Project: build logstat, an installable Node CLI that streams access logs, parses flags with util.parseArgs, reports stats, and sets exit codes."
---

# 10 · Project — A Real CLI Tool

Node is excellent for command-line tools: fast startup, a huge package ecosystem, and
cross-platform file and process APIs. In this project you build **logstat**, a CLI
that summarizes web-server access logs — requests by status class, bytes served, and the
most-hit paths — using only built-in modules. It pulls together modules, npm `bin`
entries, `fs`, streams via `readline`, async iteration, and error handling from this level.

## What you're building

```text
$ logstat access.log
requests: 7  bytes: 1299  invalid lines: 1
status classes: 2xx=3 3xx=1 4xx=2 5xx=1
top paths:
┌─────────┬──────────────────┬───────┐
│ (index) │ path             │ count │
├─────────┼──────────────────┼───────┤
│ 0       │ '/api/users'     │ 3     │
│ 1       │ '/api/login'     │ 2     │
│ 2       │ '/api/orders/42' │ 1     │
│ 3       │ '/favicon.ico'   │ 1     │
└─────────┴──────────────────┴───────┘
```

Requirements:

- Read a file argument **or** standard input (so `cat *.log | logstat` works).
- Handle files larger than memory — process line by line.
- Flags: `--top N`, `--status 5xx`, `--json`, `--help`.
- Clear error messages and conventional **exit codes**: `0` success, `1` runtime
  failure (e.g. unreadable file), `2` usage error.
- Installable as a command with `npm link` or `npm install -g`.

## Project layout

```text
logstat/
  package.json
  bin/logstat.js      # entry point: argument parsing, I/O, output
  src/parse.js        # pure: one log line -> object
  src/stats.js        # pure: accumulate and summarize
  test/parse.test.js
  access.log          # sample data
```

Keeping parsing and statistics as **pure functions** (no I/O) makes them trivial to test
and reuse; the `bin` file is the only part that touches `process`.

## Step 1: package.json with a bin entry

```json title="package.json"
{
  "name": "logstat",
  "version": "1.0.0",
  "type": "module",
  "bin": { "logstat": "./bin/logstat.js" },
  "engines": { "node": ">=22" }
}
```

`bin` maps a command name to a file. On `npm install -g` or `npm link`, npm creates a
`logstat` executable on your `PATH` pointing at that file.

## Step 2: parsing a line

```js title="src/parse.js"
// Matches the common/combined access-log format used by nginx and Apache:
// 203.0.113.9 - - [10/Oct/2025:13:55:36 +0000] "GET /api/users HTTP/1.1" 200 512 ...
const LINE = /^(\S+) \S+ \S+ \[([^\]]+)\] "(\S+) (\S+) [^"]*" (\d{3}) (\d+|-)/;

export function parseLine(line) {
  const m = LINE.exec(line);
  if (!m) return null;
  const [, ip, time, method, url, status, bytes] = m;
  return {
    ip, time, method,
    path: url.split('?')[0],
    status: Number(status),
    bytes: bytes === '-' ? 0 : Number(bytes),
  };
}
```

The regex captures the fields we need from the common/combined log format; anything that
doesn't match returns `null` so the caller can count it as invalid rather than crash.

## Step 3: accumulating statistics

```js title="src/stats.js"
export function createStats() {
  return { total: 0, invalid: 0, bytes: 0, byStatus: new Map(), byPath: new Map() };
}

export function addEntry(stats, entry) {
  stats.total++;
  stats.bytes += entry.bytes;
  const cls = `${Math.floor(entry.status / 100)}xx`;
  stats.byStatus.set(cls, (stats.byStatus.get(cls) ?? 0) + 1);
  stats.byPath.set(entry.path, (stats.byPath.get(entry.path) ?? 0) + 1);
}

export function summarize(stats, top) {
  return {
    requests: stats.total,
    invalidLines: stats.invalid,
    bytes: stats.bytes,
    status: Object.fromEntries([...stats.byStatus].sort()),
    topPaths: [...stats.byPath]
      .sort((a, b) => b[1] - a[1])
      .slice(0, top)
      .map(([path, count]) => ({ path, count })),
  };
}
```

`Map` is used instead of a plain object for counts: keys like `/constructor` or
`__proto__` in a URL would collide with object prototype properties.

## Step 4: the command

```js title="bin/logstat.js"
#!/usr/bin/env node
import { parseArgs } from 'node:util';
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';
import { parseLine } from '../src/parse.js';
import { createStats, addEntry, summarize } from '../src/stats.js';

const HELP = `Usage: logstat [options] [file]

Summarize an nginx/Apache access log. Reads stdin if no file is given.

Options:
  -t, --top <n>       number of top paths to show (default 5)
  -s, --status <cls>  only count this status class, e.g. 5xx
  -j, --json          print JSON instead of a table
  -h, --help          show this help`;

let args;
try {
  args = parseArgs({
    allowPositionals: true,
    options: {
      top: { type: 'string', short: 't', default: '5' },
      status: { type: 'string', short: 's' },
      json: { type: 'boolean', short: 'j', default: false },
      help: { type: 'boolean', short: 'h', default: false },
    },
  });
} catch (err) {
  console.error(`logstat: ${err.message}\n\n${HELP}`);
  process.exit(2);
}

const { values, positionals } = args;
if (values.help) {
  console.log(HELP);
  process.exit(0);
}

const top = Number.parseInt(values.top, 10);
if (!Number.isInteger(top) || top < 1) {
  console.error('logstat: --top must be a positive integer');
  process.exit(2);
}

const file = positionals[0];
const input = file ? createReadStream(file) : process.stdin;
input.on('error', (err) => {
  console.error(`logstat: cannot read ${file}: ${err.code ?? err.message}`);
  process.exit(1);
});

const stats = createStats();
const rl = createInterface({ input, crlfDelay: Infinity });
for await (const line of rl) {
  if (!line.trim()) continue;
  const entry = parseLine(line);
  if (!entry) { stats.invalid++; continue; }
  if (values.status && `${Math.floor(entry.status / 100)}xx` !== values.status) continue;
  addEntry(stats, entry);
}

const summary = summarize(stats, top);
if (values.json) {
  console.log(JSON.stringify(summary, null, 2));
} else {
  console.log(`requests: ${summary.requests}  bytes: ${summary.bytes}  invalid lines: ${summary.invalidLines}`);
  console.log('status classes:', Object.entries(summary.status).map(([k, v]) => `${k}=${v}`).join(' '));
  console.log('top paths:');
  console.table(summary.topPaths);
}
```

Key decisions:

- `#!/usr/bin/env node` (the shebang) lets the OS run the file directly after
  `chmod +x`. On Windows, npm generates a `.cmd` wrapper instead.
- `util.parseArgs` is Node's built-in flag parser. It rejects unknown options when
  `strict` (the default), which we turn into exit code 2 plus the help text.
- `readline.createInterface` over a stream yields one line at a time with
  `for await`, so a 5 GB log uses a few kilobytes of memory.
- The stream's `'error'` handler turns `ENOENT` into a friendly message and exit code 1.

## Step 5: sample data and a run

```text title="access.log"
203.0.113.9 - - [10/Oct/2025:13:55:36 +0000] "GET /api/users HTTP/1.1" 200 512 "-" "curl/8.0"
203.0.113.9 - - [10/Oct/2025:13:55:37 +0000] "GET /api/users?page=2 HTTP/1.1" 200 498 "-" "curl/8.0"
198.51.100.4 - - [10/Oct/2025:13:55:39 +0000] "POST /api/login HTTP/1.1" 401 64 "-" "Mozilla/5.0"
198.51.100.4 - - [10/Oct/2025:13:55:41 +0000] "POST /api/login HTTP/1.1" 200 128 "-" "Mozilla/5.0"
192.0.2.77 - - [10/Oct/2025:13:56:02 +0000] "GET /api/orders/42 HTTP/1.1" 500 97 "-" "okhttp/4.12"
192.0.2.77 - - [10/Oct/2025:13:56:03 +0000] "GET /favicon.ico HTTP/1.1" 404 - "-" "Mozilla/5.0"
this line is garbage
203.0.113.9 - - [10/Oct/2025:13:57:10 +0000] "GET /api/users HTTP/1.1" 304 0 "-" "curl/8.0"
```

```bash
chmod +x bin/logstat.js
./bin/logstat.js -s 2xx -j -t 2 access.log
```

```json
{
  "requests": 3,
  "invalidLines": 1,
  "bytes": 1138,
  "status": {
    "2xx": 3
  },
  "topPaths": [
    {
      "path": "/api/users",
      "count": 2
    },
    {
      "path": "/api/login",
      "count": 1
    }
  ]
}
```

Error paths, as observed:

```text
$ ./bin/logstat.js missing.log; echo "exit=$?"
logstat: cannot read missing.log: ENOENT
exit=1
$ ./bin/logstat.js --bogus; echo "exit=$?"
logstat: Unknown option '--bogus'. ...
(help text)
exit=2
```

And stdin works: `cat access.log | ./bin/logstat.js -t 1`.

## Step 6: tests with the built-in runner

```js title="test/parse.test.js"
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { parseLine } from '../src/parse.js';

test('parses a combined-format line', () => {
  const e = parseLine('203.0.113.9 - - [10/Oct/2025:13:55:36 +0000] "GET /api/users?page=2 HTTP/1.1" 200 512 "-" "curl/8.0"');
  assert.deepEqual(e, {
    ip: '203.0.113.9', time: '10/Oct/2025:13:55:36 +0000', method: 'GET',
    path: '/api/users', status: 200, bytes: 512,
  });
});

test('treats "-" bytes as 0', () => {
  assert.equal(parseLine('1.2.3.4 - - [x] "GET / HTTP/1.1" 404 - "-" "-"').bytes, 0);
});

test('returns null for garbage', () => {
  assert.equal(parseLine('this line is garbage'), null);
});
```

```text
$ node --test
✔ parses a combined-format line
✔ treats "-" bytes as 0
✔ returns null for garbage
ℹ tests 3
ℹ pass 3
ℹ fail 0
```

`node --test` with no arguments finds files matching common patterns such as
`*.test.js` under `test/`. Level 2 covers the test runner properly.

## Step 7: install it as a command

```bash
npm link          # symlinks this package globally; `logstat` is now on your PATH
logstat --help
npm unlink -g logstat
```

## How It Actually Works

**Line-by-line reading.** `createReadStream` reads the file in chunks (64 KiB by default)
on the libuv thread pool. `readline` listens for `'data'`, appends each chunk to an
internal string buffer, splits on `\n` (with `crlfDelay: Infinity` treating `\r\n` as one
break), and emits a `'line'` event for each complete line, keeping the trailing partial
line for the next chunk. Its async iterator buffers lines and pauses the underlying
stream while your loop body is busy, so memory stays bounded.

**stdin.** `process.stdin` is a readable stream over file descriptor 0. When you pipe
`cat access.log |`, fd 0 is a pipe; when you run interactively it's a TTY and the
program waits for you to type (end with Ctrl+D). The same code handles both because both
are streams.

**Exit codes and stdout.** `process.exit(code)` ends the process immediately; pending
asynchronous writes may be cut short. For our error paths we only exit after writing
short messages to stderr, which is synchronous for files and TTYs. When the success
path finishes naturally the loop empties and Node exits with `process.exitCode` (0 by
default) after flushing output — prefer setting `process.exitCode = 1` over calling
`process.exit()` when output might still be in flight, such as when stdout is a pipe.

**The shebang and bin.** The kernel reads the first line `#!/usr/bin/env node`, runs
`env`, which finds `node` on your `PATH` and executes it with the script path. `npm link`
creates a symlink in the global `bin` directory pointing to your file.

## Common mistakes

- **Reading the whole file with `readFile().split('\n')`** — breaks on large logs.
- **Mixing output and diagnostics.** Results go to stdout, errors to stderr, so
  `logstat x.log > report.txt` still shows errors on the terminal.
- **Exit code 0 on failure.** Scripts and CI rely on exit codes; always signal failure.
- **Forgetting `chmod +x`** or the shebang → "permission denied" or a shell trying to run
  JavaScript.
- **Putting I/O inside parsing functions**, making them hard to test.

## Exercise

1. Add `--since "10/Oct/2025:13:56"` that skips earlier entries. Parse the log timestamp
   into a `Date` properly instead of comparing strings.
2. Add a `--format csv` option for the top paths.
3. Report the 95th-percentile response size, and write a test for it in `test/stats.test.js`.
4. Generate a 1-million-line log with a small script, run `logstat` on it under
   `/usr/bin/time -v` (Linux) or `/usr/bin/time -l` (macOS), and confirm memory stays
   small. Then try a naive `readFile` version and compare.
