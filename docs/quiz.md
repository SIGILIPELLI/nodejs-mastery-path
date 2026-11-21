---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Node.js Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What is printed?\n```javascript\nimport { EventEmitter } from 'node:events'\nconst e = new EventEmitter()\ne.on('x', () => console.log('A'))\ne.once('x', () => console.log('B'))\nconsole.log('start')\nconsole.log(e.emit('x'))\nconsole.log(e.emit('x'))\nconsole.log(e.emit('nobody'), e.listenerCount('x'))\n```\n",
    "options": [
      "start, true, A, B, A, true, false 1",
      "start, A, B, true, A, B, true, false 2",
      "A, B, start, true, A, true, false 1",
      "start, A, B, true, A, true, false 1 (one per line)"
    ],
    "correctIndex": 3,
    "explanation": "emit is synchronous, so listeners run before emit returns true. The once listener is removed after the first call. emit returns false with no listeners, and one listener remains."
  },
  {
    "question": "What does this print?\n```javascript\nconsole.log(Buffer.from('héllo €').length, 'héllo €'.length, Buffer.byteLength('€'))\n```\n",
    "options": [
      "10 7 3",
      "10 10 3",
      "7 10 3",
      "7 7 1"
    ],
    "correctIndex": 0,
    "explanation": "In UTF-8, é takes 2 bytes and € takes 3, so the string has 7 characters but 10 bytes."
  },
  {
    "question": "What is printed?\n```javascript\nconst b = Buffer.from('abcdef')\nconst v = b.subarray(0, 3); v[0] = 0x5a\nconsole.log(b.toString())\nconst c = Buffer.from(b); c[0] = 0x71\nconsole.log(b.toString(), c.toString())\n```\n",
    "options": [
      "Zbcdef, then Zbcdef qbcdef",
      "Zbcdef, then qbcdef qbcdef",
      "abcdef, then abcdef qbcdef",
      "abcdef, then Zbcdef Zbcdef"
    ],
    "correctIndex": 0,
    "explanation": "subarray shares memory with the original, but Buffer.from(buffer) makes an independent copy."
  },
  {
    "question": "What does this print?\n```javascript\nprocess.env.N = 5\nconsole.log(typeof process.env.N, process.env.N, process.env.NOPE)\n```\n",
    "options": [
      "string 5 undefined",
      "string 5 null",
      "number 5 undefined",
      "number 5 null"
    ],
    "correctIndex": 0,
    "explanation": "Values in process.env are always strings (or undefined when missing)."
  },
  {
    "question": "In what order are these lines printed?\n```javascript\nasync function f() { console.log('f1'); await null; console.log('f2') }\nf()\nPromise.resolve().then(() => console.log('p1'))\nconsole.log('sync')\n```\n",
    "options": [
      "sync, f1, f2, p1",
      "f1, sync, p1, f2",
      "f1, sync, f2, p1",
      "f1, f2, sync, p1"
    ],
    "correctIndex": 2,
    "explanation": "f runs synchronously up to its first await; the continuation is queued as a microtask before the .then callback, so f2 prints before p1."
  },
  {
    "question": "What is printed?\n```javascript\nconst r = await Promise.allSettled([Promise.resolve(1), Promise.reject(new Error('no')), 3])\nconsole.log(r.map(x => x.status).join(','))\n```\n",
    "options": [
      "fulfilled,rejected",
      "fulfilled,rejected,fulfilled",
      "rejected",
      "fulfilled,fulfilled,fulfilled"
    ],
    "correctIndex": 1,
    "explanation": "allSettled never rejects: it reports each outcome. Promise.all would reject on the first failure."
  },
  {
    "question": "What does this print?\n```javascript\nconsole.log(path.join('/srv/app', 'uploads', '../logs', 'app.log'), path.basename('/a/b/c.txt', '.txt'), path.extname('x.tar.gz'))\n```\n",
    "options": [
      "/srv/app/uploads/../logs/app.log c .gz",
      "/srv/app/logs/app.log c.txt .gz",
      "/srv/app/uploads/logs/app.log c .tar.gz",
      "/srv/app/logs/app.log c .gz"
    ],
    "correctIndex": 3,
    "explanation": "path.join normalises '..', basename strips the given extension, and extname returns only the last extension."
  },
  {
    "question": "What does this print?\n```javascript\nconst u = new URL('/notes?q=Tea&q=Milk&limit=5', 'http://localhost')\nconsole.log(u.pathname, u.searchParams.get('q'), u.searchParams.getAll('q').length, Number(u.searchParams.get('limit')) + 1)\n```\n",
    "options": [
      "/notes Tea 2 6",
      "/notes Tea 1 6",
      "/notes?q=Tea 2 6",
      "/notes Milk 2 51"
    ],
    "correctIndex": 0,
    "explanation": "get returns the first value, getAll returns all of them, and query values are strings, so limit needs Number()."
  },
  {
    "question": "Which of these does the range ^5.1.0 accept, and which does ^0.4.2 accept? (5.9.1, 6.0.0 against ^5.1.0; 0.4.9, 0.5.0 against ^0.4.2)",
    "options": [
      "^5.1.0: 5.9.1 yes, 6.0.0 no. ^0.4.2: 0.4.9 yes, 0.5.0 no",
      "^5.1.0: 6.0.0 yes. ^0.4.2: 0.5.0 yes",
      "^5.1.0: 5.9.1 no. ^0.4.2: 0.4.9 no",
      "^5.1.0: both yes. ^0.4.2: both yes"
    ],
    "correctIndex": 0,
    "explanation": "Caret allows changes that don't bump the leftmost non-zero part, so for 0.x versions a minor bump may break."
  },
  {
    "question": "What does this pipeline produce?\n```javascript\nconst out = []\nawait pipeline(\n  Readable.from(['a', 'b', 'c']),\n  new Transform({ transform(ch, enc, cb) { cb(null, ch.toString().toUpperCase()) } }),\n  async function (src) { for await (const x of src) out.push(x.toString()) },\n)\nconsole.log(out.join(''))\n```\n",
    "options": [
      "a,b,c",
      "An error",
      "abc",
      "ABC"
    ],
    "correctIndex": 3,
    "explanation": "Transform maps each chunk to upper case; pipeline wires the streams and handles backpressure and errors."
  },
  {
    "question": "A server hashes passwords with scrypt and a random salt per password. For the same password, are two stored hashes equal?",
    "options": [
      "Only on the same day",
      "Yes, always",
      "Yes if the cost is the same",
      "No: a different random salt gives a different hash, so compare with the stored salt and timingSafeEqual"
    ],
    "correctIndex": 3,
    "explanation": "The same password and the same salt gives equal output (verified by running), but a fresh salt does not."
  },
  {
    "question": "What delays does exponential backoff with base 100 ms give for four retries?",
    "options": [
      "100, 200, 400, 800 ms (plus jitter in production)",
      "100, 300, 500, 700 ms",
      "100, 100, 100, 100 ms",
      "100, 1000, 10000, 100000 ms"
    ],
    "correctIndex": 0,
    "explanation": "base × 2^i. Add jitter so many clients don't retry at the same instant."
  },
  {
    "question": "A circuit breaker has threshold 3. The calls fail, fail, fail, then succeed, succeed. What are the results?\n(breaker returns 'open' while failures ≥ threshold)",
    "options": [
      "fail, fail, fail, ok, ok",
      "fail, fail, fail, fail, ok",
      "open, open, open, ok, ok",
      "fail, fail, fail, open, open"
    ],
    "correctIndex": 3,
    "explanation": "After three consecutive failures it fails fast until the cool-down, even though the dependency may have recovered (the simulated breaker has no cool-down here)."
  },
  {
    "question": "Why does a 500 ms CPU-bound loop make a 10 ms timer fire at about 500 ms?",
    "options": [
      "libuv sleeps",
      "V8 pauses timers",
      "Node runs JavaScript on one main thread, so the event loop can only check timers when the current code finishes",
      "Timers are inaccurate"
    ],
    "correctIndex": 2,
    "explanation": "Move CPU-heavy work to worker_threads."
  },
  {
    "question": "Inside an fs callback, which runs first: setImmediate or setTimeout(fn, 0)?",
    "options": [
      "Random",
      "Neither",
      "setImmediate, because the check phase follows the poll phase",
      "setTimeout"
    ],
    "correctIndex": 2,
    "explanation": "Verified by running: 'immediate inside I/O' printed before 'timeout inside I/O'."
  },
  {
    "question": "What should a service do on SIGTERM?",
    "options": [
      "Restart itself",
      "Ignore it",
      "Exit immediately",
      "Report not ready, stop accepting connections, finish in-flight requests, close resources, then exit before SIGKILL"
    ],
    "correctIndex": 3,
    "explanation": "SIGKILL can't be caught, so use a hard deadline timer."
  },
  {
    "question": "Why should file uploads be sniffed by bytes and renamed on disk?",
    "options": [
      "It saves disk space",
      "It makes uploads faster",
      "Multer requires it",
      "The client's mimetype, extension and filename can lie or contain ../ paths; use magic numbers and a generated name"
    ],
    "correctIndex": 3,
    "explanation": "Also set size limits."
  },
  {
    "question": "How does Express recognise an error-handling middleware, and where is it registered?",
    "options": [
      "It is named errorHandler and registered first",
      "It has two parameters and runs before routes",
      "It returns a promise",
      "It takes four parameters (err, req, res, next) and is registered last"
    ],
    "correctIndex": 3,
    "explanation": "Express 5 also forwards rejected promises from async handlers to it."
  },
  {
    "question": "A user asks for GET /tasks/17 owned by someone else. What should the API return, and why?",
    "options": [
      "500",
      "403 with the owner's name",
      "200 with empty data",
      "404, so it doesn't reveal that the task exists, with the query scoped by owner"
    ],
    "correctIndex": 3,
    "explanation": "Scoping every query by the owner also prevents broken access control."
  },
  {
    "question": "You run three cluster workers and keep a Map as a rate-limit counter in each. What goes wrong?",
    "options": [
      "Each worker has its own heap, so counters aren't shared; use a shared store such as Redis",
      "Nothing, memory is shared",
      "The Map is faster",
      "Workers crash"
    ],
    "correctIndex": 0,
    "explanation": "Clustering doesn't give shared memory."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Retry with exponential backoff

Write an async function retry(fn, { attempts, baseMs }) that calls fn(i). If it throws, wait baseMs * 2**i ms (not after the last attempt) and try again. After the last attempt rethrow the last error. Track the waits and print them for a function that fails twice and then succeeds.

**Starter code**

```python
import { setTimeout as sleep } from 'node:timers/promises'

export async function retry(fn, { attempts = 3, baseMs = 10, onWait = () => {} } = {}) {
  // your code
}

const waits = []
let calls = 0
const result = await retry(async () => {
  if (++calls < 3) throw new Error('flaky')
  return 'finally'
}, { attempts: 4, baseMs: 10, onWait: (ms) => waits.push(ms) })
console.log(result, calls, waits.join(','))
```

**Expected output**

```text
finally 3 10,20
```

<details><summary>Show solution and explanation</summary>

<pre><code>import { setTimeout as sleep } from &#x27;node:timers/promises&#x27;

export async function retry(fn, { attempts = 3, baseMs = 10, onWait = () =&gt; {} } = {}) {
  let lastErr
  for (let i = 0; i &lt; attempts; i++) {
    try {
      return await fn(i)
    } catch (err) {
      lastErr = err
      if (i &lt; attempts - 1) {
        const ms = baseMs * 2 ** i
        onWait(ms)
        await sleep(ms)
      }
    }
  }
  throw lastErr
}

const waits = []
let calls = 0
const result = await retry(async () =&gt; {
  if (++calls &lt; 3) throw new Error(&#x27;flaky&#x27;)
  return &#x27;finally&#x27;
}, { attempts: 4, baseMs: 10, onWait: (ms) =&gt; waits.push(ms) })
console.log(result, calls, waits.join(&#x27;,&#x27;))
</code></pre>

<p>Waits grow as base × 2^i, and the last attempt doesn&#x27;t wait. Verified by running the solution in Node: output matches.</p>
</details>

### Parse access-log lines

Write parseLine(line) that matches the common log format and returns { method, path, status, bytes } or null for invalid lines. Then count requests by status class (2xx, 4xx…) for the sample lines.

**Starter code**

```python
const LINE = /^(\S+) \S+ \S+ \[([^\]]+)\] "(\S+) (\S+) [^"]*" (\d{3}) (\d+|-)/

export function parseLine(line) {
  // your code
}

const lines = [
  '203.0.113.9 - - [10/Oct/2025:13:55:36 +0000] "GET /api/users HTTP/1.1" 200 512',
  '203.0.113.9 - - [10/Oct/2025:13:55:37 +0000] "POST /api/login HTTP/1.1" 401 64',
  'garbage line',
  '198.51.100.7 - - [10/Oct/2025:13:55:38 +0000] "GET /missing HTTP/1.1" 404 -',
]
const classes = {}
let invalid = 0
for (const l of lines) {
  const p = parseLine(l)
  if (!p) { invalid++; continue }
  const k = Math.floor(p.status / 100) + 'xx'
  classes[k] = (classes[k] ?? 0) + 1
}
console.log(JSON.stringify(classes), 'invalid:', invalid)
```

**Expected output**

```text
{"2xx":1,"4xx":2} invalid: 1
```

<details><summary>Show solution and explanation</summary>

<pre><code>const LINE = /^(\S+) \S+ \S+ \[([^\]]+)\] &quot;(\S+) (\S+) [^&quot;]*&quot; (\d{3}) (\d+|-)/

export function parseLine(line) {
  const m = LINE.exec(line)
  if (!m) return null
  return { method: m[3], path: m[4], status: Number(m[5]), bytes: m[6] === &#x27;-&#x27; ? 0 : Number(m[6]) }
}

const lines = [
  &#x27;203.0.113.9 - - [10/Oct/2025:13:55:36 +0000] &quot;GET /api/users HTTP/1.1&quot; 200 512&#x27;,
  &#x27;203.0.113.9 - - [10/Oct/2025:13:55:37 +0000] &quot;POST /api/login HTTP/1.1&quot; 401 64&#x27;,
  &#x27;garbage line&#x27;,
  &#x27;198.51.100.7 - - [10/Oct/2025:13:55:38 +0000] &quot;GET /missing HTTP/1.1&quot; 404 -&#x27;,
]
const classes = {}
let invalid = 0
for (const l of lines) {
  const p = parseLine(l)
  if (!p) { invalid++; continue }
  const k = Math.floor(p.status / 100) + &#x27;xx&#x27;
  classes[k] = (classes[k] ?? 0) + 1
}
console.log(JSON.stringify(classes), &#x27;invalid:&#x27;, invalid)
</code></pre>

<p>Keeping the parser a pure function makes it easy to test. A &#x27;-&#x27; byte count becomes 0. Verified by running the solution in Node: output matches.</p>
</details>
