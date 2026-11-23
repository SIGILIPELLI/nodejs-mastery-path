---
title: "Node.js Interview Questions and Answers (180 Questions)"
description: "180 Node.js interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Node.js Mastery Path."
---

# Node.js Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Node.js Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Node.js Is: V8 + libuv](level-1/01-what-is-node.md)

**Q1.** What is Node.js, in the lesson's framing?

**Answer:** A runtime that runs JavaScript outside a browser, with built-in modules for the OS

It's the node executable plus built-in modules for files, sockets, processes, timers and cryptography.

**Q2.** What does V8 do, and what does libuv do?

**Answer:** V8 executes JavaScript; libuv provides the event loop and asynchronous OS access

V8 knows nothing about files or networks.

**Q3.** A setTimeout of 10 ms is queued, then a 500 ms busy loop runs. Roughly when does the timer fire?

**Answer:** After the loop, around 504 ms, because the thread was busy

The event loop only checks timers once the current JavaScript finishes.

**Q4.** What is the 'good consequence' of one main thread?

**Answer:** No locks or data races on your objects

The bad one: a long-running function makes everything else wait.

### [02 · Installing Node &amp; Managing Versions](level-1/02-installing-node-nvm.md)

**Q5.** Which Node major versions become LTS?

**Answer:** Even-numbered ones, promoted in October

Odd-numbered majors are Current only and are abandoned after a few months.

**Q6.** What does `node -p "process.release.lts"` print on a Current build?

**Answer:** undefined (on LTS builds it prints the codename)

It reveals whether you're running an LTS build.

**Q7.** Why is a version manager recommended over the official installer?

**Answer:** It installs versions in your home directory and switches per shell or project, with no sudo

The installer is fine for a single version, but upgrades mean reinstalling and global packages may need sudo.

**Q8.** What should you use for servers and anything you deploy?

**Answer:** An Active or Maintenance LTS version

Check github.com/nodejs/release for the schedule.

### [03 · Modules: ES Modules &amp; CommonJS](level-1/03-modules-esm-cjs.md)

**Q9.** How does Node decide a .js file's module type?

**Answer:** The nearest package.json: "type": "module" means ESM; commonjs or no type means CommonJS

*.mjs is always ESM and *.cjs always CommonJS.

**Q10.** What must relative imports include in ESM?

**Answer:** The file extension, e.g. './math.js'

ESM resolution doesn't guess extensions or look for index.js.

**Q11.** What replaces __dirname and __filename in ESM?

**Answer:** import.meta.dirname and import.meta.filename, or import.meta.url with new URL()

require is also unavailable by default.

**Q12.** What does a dynamic import('./math.mjs') return?

**Answer:** A promise

Top-level await also works in ESM.

### [04 · npm &amp; package.json](level-1/04-npm-package-json.md)

**Q13.** What does ^5.1.0 accept?

**Answer:** &gt;=5.1.0 &lt;6.0.0

~5.1.0 accepts patches only; for 0.x versions, caret is stricter (^0.4.2 means &gt;=0.4.2 &lt;0.5.0).

**Q14.** How does `npm ci` differ from `npm install`?

**Answer:** It installs exactly what the lockfile says, deletes node_modules first, and fails if package.json and the lockfile disagree

Use it in CI and Docker builds for reproducibility.

**Q15.** What does "private": true prevent?

**Answer:** An accidental npm publish of an application

It's typical for applications.

**Q16.** Why have devDependencies?

**Answer:** They're only needed to develop, test or build, and `npm ci --omit=dev` skips them, shrinking container images

dependencies are needed at runtime.

### [05 · Files &amp; Paths: fs and path](level-1/05-fs-and-path.md)

**Q17.** Which fs style is the default for application code and servers?

**Answer:** fs/promises with await

*Sync suits startup code, CLIs and build scripts, where blocking is harmless.

**Q18.** Why prefer attempting the operation and handling errors over checking first with access?

**Answer:** Another process can change the file between the check and the use (a TOCTOU race)

Handle errors by code, such as ENOENT, and rethrow real problems like EACCES.

**Q19.** What does mkdir('data/cache', { recursive: true }) do?

**Answer:** Like mkdir -p: creates parents and doesn't error if it exists

rm with recursive and force is like rm -rf — careful.

**Q20.** When should you use createReadStream?

**Answer:** For large files, or to start processing before the whole file is in memory

readFile loads the whole file.

### [06 · Async Patterns: Callbacks to async/await](level-1/06-async-patterns.md)

**Q21.** What is the error-first callback convention?

**Answer:** The last argument is a callback whose first parameter is an error (or null)

For example fs.readFile(file, enc, (err, text) =&gt; ...).

**Q22.** Why can't a try/catch around the original call catch an exception thrown inside a callback?

**Answer:** The stack has already unwound by the time the callback runs

That's one of the problems with callbacks, alongside the callback pyramid.

**Q23.** What does util.promisify do?

**Answer:** Converts an error-first callback function into a promise-returning one

Many built-ins also have promise versions such as node:fs/promises.

**Q24.** Why wrap errors with new Error(msg, { cause: err })?

**Answer:** It preserves the original root cause for logs

Use it when rethrowing with more context.

### [07 · Events &amp; EventEmitter](level-1/07-events-eventemitter.md)

**Q25.** In the order queue example, when does `added #1` print relative to `before add` and `after add`?

**Answer:** Between them, because emit is synchronous

Listeners run synchronously inside emit.

**Q26.** What happens if an EventEmitter emits 'error' with no listener?

**Answer:** The error is thrown and can crash the process

That's why the example registers q.on('error', …) — the 'error' event is special.

**Q27.** Why did 'queue full - flushing' print only once even though the queue stayed full?

**Answer:** It was registered with once, which removes the listener after one call

The listener count for 'added' stayed at 1.

**Q28.** What does events.once(emitter, name) return?

**Answer:** A promise for the next event, resolving to an array of its arguments

It's the cleanest way to await something like a 'ready' event.

### [08 · A Raw HTTP Server with node:http](level-1/08-raw-http-server.md)

**Q29.** What are req and res in node:http?

**Answer:** req is a readable stream (IncomingMessage); res is a writable stream (ServerResponse)

req.url holds the path and query string only.

**Q30.** What happens if you forget res.end()?

**Answer:** The client hangs until it times out

Always end every response path.

**Q31.** Why does readJsonBody cap the body size at MAX_BODY?

**Answer:** To reject oversized bodies with 413 instead of buffering unbounded input

Invalid JSON returns 400.

**Q32.** How is the query string read in the example?

**Answer:** With new URL(req.url, base) and url.searchParams.get('q')

The pathname is matched with regexes such as /^\/notes\/(\d+)$/.

### [09 · Environment Variables &amp; Configuration](level-1/09-env-config.md)

**Q33.** What type are process.env values?

**Answer:** Always strings or undefined

process.env.X = 1 stores '1', and assigning undefined stores 'undefined'.

**Q34.** Which takes precedence: a variable in the real environment or one in the .env file?

**Answer:** The real environment

On a test run, PORT=5000 node --env-file=.env app.js used 5000, not the file's 4000.

**Q35.** How do current Node releases load .env files?

**Answer:** Natively with --env-file or process.loadEnvFile(), no dotenv needed

--env-file-if-exists is available for optional files.

**Q36.** Why centralise and validate configuration in one module?

**Answer:** A typo then fails at startup instead of giving undefined at 3 a.m.

It also shows what a service needs at a glance.

### [10 · Project — A Real CLI Tool](level-1/10-project-cli-tool.md)

**Q37.** What do logstat's exit codes mean?

**Answer:** 0 success, 1 runtime failure (e.g. unreadable file), 2 usage error

Conventional exit codes make a CLI scriptable.

**Q38.** How does logstat handle files larger than memory?

**Answer:** It processes line by line via readline and async iteration

It also reads standard input when no file is given, so cat *.log | logstat works.

**Q39.** What does the bin entry in package.json do?

**Answer:** Maps a command name to a file; npm link or npm install -g puts an executable on your PATH

The entry is { "logstat": "./bin/logstat.js" }.

**Q40.** Why keep parsing and statistics as pure functions?

**Answer:** They're easy to test and reuse; only the bin file touches process

parse.js and stats.js do no I/O.

## Level 2 · Intermediate

### [01 · Express Fundamentals](level-2/01-express-fundamentals.md)

**Q41.** Which Express 5 change affects async handlers?

**Answer:** Rejected promises from async handlers are forwarded to error handlers automatically

Another change: req.body is undefined, not {}, when no body parser ran.

**Q42.** What does express.json() replace from the raw http version?

**Answer:** readJsonBody (with a default 100 KB limit)

res.json replaces sendJson, and app.get replaces the if chain.

**Q43.** What type are values in req.params?

**Answer:** Always strings

/books/:id gives { id: '42' }.

**Q44.** What is in req.body without a body-parsing middleware?

**Answer:** undefined

Mount express.json() to parse JSON bodies.

### [02 · Routing &amp; Middleware](level-2/02-routing-middleware.md)

**Q45.** What distinguishes middleware from a route handler?

**Answer:** Both are (req, res, next); middleware calls next() to pass control on

Auth, logging, CORS, rate limiting and body parsing all look the same: functions in a pipeline.

**Q46.** In what order does middleware run?

**Answer:** In the order registered, only for matching mount paths

The lesson's order: helmet, json, logger, /admin guard, routers, 404, error handler.

**Q47.** What is the shape of Express error-handling middleware?

**Answer:** Four arguments (err, req, res, next), registered last

The notFoundHandler comes just before it.

**Q48.** What changed in Express 5 route paths?

**Answer:** Wildcards must be named, like /files/*filepath, and regex characters are no longer special

Old '/files/*' or '/:id?' forms need updating; use {/:section} for optional segments.

### [03 · REST API Design](level-2/03-rest-api-design.md)

**Q49.** Which pair is correct for creating a resource?

**Answer:** POST /tasks returns 201 with a Location header

Model nouns as resources and let HTTP methods be the verbs.

**Q50.** Which methods are idempotent?

**Answer:** GET, PUT and DELETE

POST is neither safe nor idempotent, so retrying a create-payment POST is dangerous without idempotency keys.

**Q51.** What is the difference between 401 and 403?

**Answer:** 401: not authenticated; 403: authenticated but not allowed

404 means no such resource.

**Q52.** What is the difference between PATCH and PUT?

**Answer:** PATCH sends only the fields to change; PUT sends the whole representation

With PUT, missing fields mean remove or reset.

### [04 · Validating Input with Zod](level-2/04-validation.md)

**Q53.** Why validate input at the edge?

**Answer:** So the rest of the code can trust its inputs' shape, or the client gets a clear 400

Anything arriving over HTTP is untrusted, even an added "isAdmin": true.

**Q54.** What did CreateUser.parse({ email, name: '  Ada ', age: '36' }) return?

**Answer:** A new object: name trimmed to 'Ada', age coerced to 36, defaults added (role 'member', tags [])

Use the parsed output, not the original input.

**Q55.** How does safeParse differ from parse?

**Answer:** It returns { success, data } or { success: false, error } instead of throwing a ZodError

It's easier to turn into an HTTP response.

**Q56.** Why does a Zod schema count as 'one declaration, three benefits'?

**Answer:** Runtime validation, parsing (coercion, trimming, defaults), and a static TypeScript type

Alternatives include Valibot, ArkType, Joi, Yup and Ajv.

### [05 · Error Handling](level-2/05-error-handling.md)

**Q57.** What is an operational error?

**Answer:** An expected failure of a correct program, such as bad JSON, a missing record or a dropped DB connection

You handle these with a 4xx, a retry or degraded behaviour.

**Q58.** What should you do for a programmer error (a bug) outside any request?

**Answer:** Let the process crash and be restarted, since state may be inconsistent

For a bug inside a request, return 500, log loudly, and fix the code.

**Q59.** What does the errorHandler return to the client for status ≥ 500?

**Answer:** A generic 'internal server error', not the real message

The details go to the log, not the client.

**Q60.** Why check res.headersSent in the error handler?

**Answer:** If a response has already started, delegate to the default handler with next(err)

You can't send a second response.

### [06 · PostgreSQL with Kysely](level-2/06-databases-postgres.md)

**Q61.** What do pg and Kysely each do?

**Answer:** pg is the driver (wire protocol and connection pool); Kysely is a type-safe query builder on top

Kysely compiles to parameterized SQL and keeps you close to SQL.

**Q62.** How many Kysely instances (pools) should a process create?

**Answer:** One per process at startup, with db.destroy() on shutdown

Pass it to your routers.

**Q63.** What are migrations?

**Answer:** Versioned scripts applied in order and recorded in a table so each runs once

Kysely's Migrator runs files exporting up (and optionally down).

**Q64.** What is PGlite used for in the lesson?

**Answer:** A WebAssembly build of Postgres inside Node, handy for tests

It's used for the Level 2 project's tests.

### [07 · Authentication: Sessions, JWT &amp; Password Hashing](level-2/07-authentication.md)

**Q65.** Why not store a plain fast hash like SHA-256 of a password?

**Answer:** Attackers can guess billions of candidates per second on GPUs after a leak

Use a deliberately slow, salted, memory-hard function: argon2id, scrypt or bcrypt.

**Q66.** Why does hashing the same password twice give different results?

**Answer:** A new random salt is generated each time

The lesson shows (await hashPassword(p)) === stored is false.

**Q67.** Why use timingSafeEqual when comparing hashes?

**Answer:** To avoid leaking information via comparison timing

Cost parameters are stored with each hash so they can be raised later.

**Q68.** What is the difference between authentication and authorization?

**Answer:** Authentication is who is making the request; authorization is whether they may do that

Authorization is usually plain code like where user_id = req.user.id.

### [08 · Testing with node:test &amp; Supertest](level-2/08-testing.md)

**Q69.** What does the built-in node:test runner need?

**Answer:** No test framework dependency at all (with node:assert)

Add Supertest for HTTP-level tests without opening a real port.

**Q70.** Which test layer gives the best confidence per test for an API?

**Answer:** Integration tests through HTTP, covering routing, validation, middleware, SQL and serialization together

Keep unit tests for logic with many edge cases.

**Q71.** In the retry test where the function fails twice then succeeds, how many calls are made?

**Answer:** 3

callCount() is 3 and the result is 'finally'.

**Q72.** What does assert.rejects check?

**Answer:** That an async function rejects with the expected error

The give-up test expects the last error to be rethrown.

### [09 · Structured Logging with Pino](level-2/09-logging.md)

**Q73.** Why use structured JSON logs rather than console.log strings in production?

**Answer:** Machines can filter and aggregate fields, e.g. every error for request b7e1c2

One JSON object per line with consistent fields.

**Q74.** What is the Pino call signature?

**Answer:** logger.level(mergingObject, message), with data in the object and a constant message

A constant message ('order placed') lets you count occurrences, while orderId stays queryable.

**Q75.** What does the redact option do?

**Answer:** Replaces sensitive paths such as *.password with [redacted] in log output

The example censors req.headers.authorization, *.password and card.number.

**Q76.** At level 'info', what happens to logger.debug(...)?

**Answer:** It's dropped; levels are numbers (debug 20, info 30)

trace 10, debug 20, info 30, warn 40, error 50, fatal 60.

### [10 · Project — A REST API with Auth](level-2/10-project-rest-api-auth.md)

**Q77.** What should the API return when a user requests another user's task?

**Answer:** 404, not 403, so it doesn't reveal that the task exists

Users can never read or modify another user's tasks.

**Q78.** What are the project's lifetime and format for access tokens?

**Answer:** A JWT access token valid for 15 minutes, sent as Authorization: Bearer &lt;token&gt;

Registration and login return it.

**Q79.** Which status codes does the project use for validation errors, duplicates and unknown routes?

**Answer:** 400 with per-field details, 409, and a JSON 404

Bugs return a 500 without internals.

**Q80.** How does the test suite run against a real Postgres engine without a server?

**Answer:** PGlite via a test-only in-process pool adapter

It's driven with Supertest.

## Level 3 · Advanced

### [01 · The Event Loop in Depth](level-3/01-event-loop-deep-dive.md)

**Q81.** Which two queues are drained after every single callback, in every phase?

**Answer:** The process.nextTick queue, then the microtask queue

The loop advances only when both are empty.

**Q82.** Inside an I/O (fs) callback, which runs first: setImmediate or setTimeout(fn, 0)?

**Answer:** setImmediate, because the check phase follows the poll phase

Verified: 'immediate inside I/O' printed before 'timeout inside I/O'. From the main module their order can vary.

**Q83.** Why could a recursive process.nextTick freeze a server?

**Answer:** The nextTick queue is drained completely before the loop proceeds, so it starves I/O

The same applies to microtasks queuing microtasks.

**Q84.** In which phase do setImmediate callbacks run?

**Answer:** Check, right after poll

Timers handle setTimeout and setInterval; poll waits for I/O.

### [02 · Streams &amp; Backpressure](level-3/02-streams-backpressure.md)

**Q85.** What does write() returning false mean?

**Answer:** The internal buffer is full; stop until the 'drain' event

It's advisory: nothing forces you to stop, but ignoring it piles data up in memory.

**Q86.** What does highWaterMark control?

**Answer:** How many bytes can be buffered before write() returns false

It bounds memory regardless of how slow the consumer is.

**Q87.** Which stream type is zlib.createGzip()?

**Answer:** Transform

Transform streams derive output from input; Duplex has independent sides, like a TCP socket.

**Q88.** What helper handles backpressure for you so you rarely write the drain loop by hand?

**Answer:** pipeline

The hand-written loop shows how it works.

### [03 · Buffers &amp; Binary Data](level-3/03-buffers.md)

**Q89.** Buffer.from('héllo €', 'utf8') has what length versus the string's length?

**Answer:** 10 bytes versus 7 characters

é is 2 bytes and € is 3 bytes. Use Buffer.byteLength(str) for Content-Length, never str.length.

**Q90.** Does Buffer.subarray copy memory?

**Answer:** No: it shares memory, so changing the view changes the original

Buffer.from(buffer) makes an independent copy.

**Q91.** What goes wrong when a chunk boundary falls inside the multi-byte character €?

**Answer:** toString() on each chunk yields replacement characters; StringDecoder holds back incomplete sequences

setEncoding('utf8') on a stream does this internally.

**Q92.** What is base64url used for?

**Answer:** URL-safe base64 without padding, as in JWTs

Buffer is a subclass of Uint8Array.

### [04 · Worker Threads &amp; Child Processes](level-3/04-worker-threads-child-processes.md)

**Q93.** When do you use worker_threads versus child_process?

**Answer:** worker_threads for CPU-bound JavaScript in the same process; child_process for external programs or strong isolation

Each worker has its own V8 isolate and event loop.

**Q94.** Why does heavy computation on the main thread hurt a server?

**Answer:** It blocks the event loop, so every other request waits

The lesson's heartbeat counter shows the main thread stalling.

**Q95.** How do the main thread and a worker communicate?

**Answer:** By messages: parentPort.on('message') and postMessage

The worker posts { id, result } back.

**Q96.** What do four workers running four jobs achieve?

**Answer:** True parallelism across CPU cores while the main thread stays free

The comparison measured heartbeats during each run.

### [05 · Clustering &amp; Using Every Core](level-3/05-clustering.md)

**Q97.** Why run several copies of a Node process?

**Answer:** One process uses one core, so an 8-core server stays mostly idle once request handling is the bottleneck

Two ways: node:cluster, or N replicas behind a load balancer as containers and Kubernetes do.

**Q98.** How are requests distributed in the lesson's three-worker demo?

**Answer:** Round-robin across workers, and the primary replaces a worker that dies

The port stays open throughout because the primary owns it.

**Q99.** What does clustering not give you?

**Answer:** Shared memory: each worker has its own heap, so in-memory caches, rate-limit counters and sessions exist per worker

Use Redis or another shared store.

**Q100.** What does worker.disconnect() on SIGTERM do?

**Answer:** Lets workers finish in-flight requests, then exit

exitedAfterDisconnect tells the primary it was intentional, so it doesn't fork a replacement.

### [06 · Caching with Redis](level-3/06-caching-redis.md)

**Q101.** What is the cache-aside pattern?

**Answer:** On read, try the cache, load from the database on a miss and store with a TTL; on write, update the database and delete the cache key

Deleting is safer than updating the cached value.

**Q102.** What is the downside of in-process caches compared with Redis?

**Answer:** Each process has its own copy, it's lost on restart, and it competes with your heap

Redis is shared by all processes and survives app restarts.

**Q103.** Why must the node-redis client have an 'error' listener?

**Answer:** It's an EventEmitter, and an unhandled 'error' would crash the process on a Redis blip

The client reconnects automatically.

**Q104.** What does the lesson say about the Redis calls it shows?

**Answer:** They were checked against the package's type definitions but not run; the cache logic was run against an in-memory stand-in

The lesson is honest about what was verified.

### [07 · WebSockets](level-3/07-websockets.md)

**Q105.** What do WebSockets provide beyond HTTP?

**Answer:** A long-lived full-duplex connection where either side can send at any time

Useful for chat, live dashboards and multiplayer games.

**Q106.** When is Server-Sent Events (SSE) enough?

**Answer:** For one-way server-to-client streams like notifications and dashboards, with browser auto-reconnect

It's a plain HTTP response that stays open.

**Q107.** What does the heartbeat with ping and terminate do?

**Answer:** Detects and terminates connections that stop answering (dead NAT paths, sleeping laptops)

A socket is marked not alive until it answers pong.

**Q108.** Why does the server set maxPayload and close with 1007 for invalid JSON?

**Answer:** To bound memory per message and reject malformed input

Binary frames are closed with 1003 'text only'.

### [08 · File Uploads](level-3/08-file-uploads.md)

**Q109.** Why not use the client's filename on disk?

**Answer:** It may be something like ../../app/server.js; generate your own name (e.g. randomUUID)

The diskStorage filename callback returns randomUUID().

**Q110.** How should you check an uploaded file's type?

**Answer:** Check the actual bytes (magic-number signatures), not the client-supplied mimetype or extension

For example PNG starts with 89 50 4e 47 and JPEG with ff d8 ff.

**Q111.** Why doesn't express.json() work for file uploads?

**Answer:** Multipart bodies need a streaming multipart parser such as multer (built on busboy)

The body has parts separated by a boundary string.

**Q112.** Which limits does the example set?

**Answer:** fileSize 2 MB, one file and five fields

Limits prevent enormous uploads; production systems often upload directly to object storage.

### [09 · Security for Node Services](level-3/09-security.md)

**Q113.** What is broken access control in a Node API?

**Answer:** GET /tasks/:id returning any user's task; defend by scoping every query by owner

It's the first OWASP category.

**Q114.** What does helmet() do?

**Answer:** Sets security headers such as content-security-policy and x-content-type-options

The lesson checks that the headers appear on a response.

**Q115.** What does rateLimit({ windowMs: 60_000, limit: 5 }) on /auth/login do?

**Answer:** Allows 5 requests per minute per client, slowing credential guessing

No rate limit on login is an 'insecure design' example.

**Q116.** How do you defend against SSRF, where the server fetches a user-supplied URL?

**Answer:** URL allowlists

Also use execFile rather than exec for injection defense.

### [10 · Project — A Real-Time Chat App](level-3/10-project-realtime-app.md)

**Q117.** How does the chat server authenticate connections?

**Answer:** During the upgrade, with a signed expiring token and an Origin check

Tokens are short-lived and issued by the Express app.

**Q118.** How are client messages validated?

**Answer:** With a Zod discriminated union keyed by type (join, leave, say, typing)

Anything else is rejected before it reaches the room logic.

**Q119.** What protects against slow consumers?

**Answer:** Checking bufferedAmount (streams and backpressure)

Each connection also has a token-bucket rate limit.

**Q120.** Why does the Rooms state need pub/sub to scale out?

**Answer:** It's in-memory for one process, so other processes wouldn't see it

Graceful shutdown uses close code 1001.

## Level 4 · Master

### [01 · Performance Profiling &amp; Memory Leaks](level-4/01-profiling-memory-leaks.md)

**Q121.** What is the most time-saving rule of performance work?

**Answer:** Profile before optimizing; the slow part is rarely where you expect

Guessing at causes wastes days.

**Q122.** How does V8's sampling CPU profiler work?

**Answer:** Roughly every millisecond it records the current JavaScript call stack; functions in many samples are where CPU time goes

Collect with node --cpu-prof, or --inspect with Chrome DevTools.

**Q123.** What does `node --cpu-prof` write?

**Answer:** A .cpuprofile file when the process exits

It's JSON, viewable as a flame chart in DevTools.

**Q124.** What were the two hidden inefficiencies in buildReport?

**Answer:** Re-sorting the whole array for every order, and JSON round-trips to copy objects

The report of 3000 orders took about 910 ms.

### [02 · Diagnostics in Production](level-4/02-diagnostics.md)

**Q125.** What is a Node diagnostic report?

**Answer:** A JSON snapshot of the process: versions, stacks, heap stats, resource usage and every libuv handle

process.report.getReport() generates one on demand.

**Q126.** How do you get a report from a hung process without killing it?

**Answer:** Run with --report-on-signal and send SIGUSR2: kill -USR2 &lt;pid&gt;

The libuv section then shows open sockets and timers.

**Q127.** What did the lesson find when --report-directory pointed to a missing directory?

**Answer:** Node printed 'Failed to open Node.js report file' and wrote nothing

The directory must already exist.

**Q128.** Why treat diagnostic reports as sensitive?

**Answer:** They contain environment variables, potentially secrets

Newer versions can exclude them with --report-exclude-env.

### [03 · TypeScript for Node](level-4/03-typescript-for-node.md)

**Q129.** What two separate jobs does TypeScript tooling do?

**Answer:** Type checking (tsc) and producing runnable JavaScript (removing annotations)

Current Node can strip types itself when you run node file.ts; type checking remains tsc's job.

**Q130.** Why must imports use './money.ts' with the extension when running .ts directly?

**Answer:** Node resolves the file exactly as written (ESM rules)

The type modifier (import { type Money }) marks an import as erased at runtime.

**Q131.** What does `node src/main.ts` print for add($19.99, $5.00) in the lesson?

**Answer:** $24.99

Amounts are integer minor units: 1999 + 500 = 2499 cents.

**Q132.** Why store money as integer minor units?

**Answer:** To avoid floating-point errors with currency

The format function divides by 100 for display.

### [04 · Architecture: Layered &amp; Hexagonal](level-4/04-architecture.md)

**Q133.** In the layered architecture, what belongs in route handlers?

**Answer:** HTTP translation only: parse and validate input, call a service, shape the response

Services hold business rules with no req/res; repositories hold data access.

**Q134.** Why should services take plain arguments and return plain values?

**Answer:** So a CLI, a queue worker or a test can call them too

They contain no req or res.

**Q135.** In hexagonal architecture, who defines ports and who implements them?

**Answer:** The core defines ports (interfaces it needs); adapters implement them

Dependency arrows all point inward.

**Q136.** What does this make easier?

**Answer:** Testing the core with in-memory adapters, and swapping infrastructure

The core imports nothing from Express, Kysely or an SDK.

### [05 · Queues &amp; Background Jobs](level-4/05-queues-background-jobs.md)

**Q137.** Why move slow work (emails, PDFs, image resizing) out of the request?

**Answer:** Doing it inline slows responses, ties latency to third-party reliability, and loses work on a restart

The request enqueues a job and returns, often 202 Accepted.

**Q138.** What does FOR UPDATE SKIP LOCKED do in the Postgres queue?

**Answer:** Lets multiple workers claim different jobs without waiting on rows another worker has locked

pg-boss and Graphile Worker build on the same technique.

**Q139.** Which is the most widely used Node job library on Redis?

**Answer:** BullMQ

It offers delays, retries, priorities, rate limits and repeatable jobs.

**Q140.** What happens to a job that keeps failing past max_attempts?

**Answer:** It ends in a 'dead' status for inspection

Statuses: queued, running, done, dead.

### [06 · Microservices Communication](level-4/06-microservices-communication.md)

**Q141.** What must every service-to-service HTTP call have?

**Answer:** A timeout, so a hung dependency doesn't hang you

Also bounded retries with exponential backoff and jitter, a circuit breaker and propagated request ids.

**Q142.** What does the lesson's circuit breaker do after `threshold` consecutive failures?

**Answer:** Fails fast for coolDownMs, then allows one trial request (half-open)

It stops a dead dependency from consuming resources.

**Q143.** Which statuses are treated as retryable?

**Answer:** 502, 503 and 504 (transient failures)

Only transient failures should be retried.

**Q144.** What does the lesson caution about microservices?

**Answer:** Many teams are better served by a modular monolith until organisational or scaling pressure justifies splitting

Every in-process call becomes a network call that can be slow or fail.

### [07 · Containerizing &amp; Deploying](level-4/07-containerize-deploy.md)

**Q145.** Why copy package manifests before the source in a Dockerfile?

**Answer:** So the npm ci layer is cached and only re-runs when dependencies change

npm ci --omit=dev installs exactly the lockfile without test tools.

**Q146.** Why USER node?

**Answer:** A compromised process shouldn't be root inside the container

Official images include an unprivileged node user.

**Q147.** What does the lesson honestly say about its Dockerfile?

**Answer:** It was written for Docker but not built or run, because the authoring machine had no container runtime

Build it yourself and fix anything your environment disagrees with.

**Q148.** How do -slim and -alpine images differ?

**Answer:** -slim is Debian-based without compilers; -alpine is smaller but uses musl libc, which some native modules don't support well

Distroless images are smaller still, with no shell.

### [08 · Observability with OpenTelemetry](level-4/08-observability.md)

**Q149.** What is the difference between monitoring and observability?

**Answer:** Monitoring tells you that something is wrong; observability lets you ask why without shipping new code

It rests on logs, metrics and traces.

**Q150.** What do traces show?

**Answer:** The path of one request through services as a tree of timed spans

They answer 'which of the 14 calls behind this slow request was slow?'.

**Q151.** What is OpenTelemetry?

**Answer:** A vendor-neutral standard for producing logs, metrics and traces, exported over OTLP to a collector

Route data to Jaeger, Grafana, Honeycomb, Datadog and so on.

**Q152.** Why does instrumentation live in its own file loaded with --import?

**Answer:** It must load before the app imports the modules it patches, and ESM needs a loader hook via register()

For CommonJS require it's easy to intercept; ESM needs the hook.

### [09 · Graceful Shutdown &amp; Reliability](level-4/09-graceful-shutdown-reliability.md)

**Q153.** What does a platform send when it stops an instance, and what can't be caught?

**Answer:** SIGTERM, then after a grace period SIGKILL, which can't be caught

A good service finishes within the grace period.

**Q154.** What is the first step of the shutdown sequence?

**Answer:** Stop being ready (e.g. /readyz returns 503) so the load balancer stops sending new requests

Then server.close(), finish in-flight work, close idle keep-alives, close resources in order, and exit.

**Q155.** Why set a hard deadline timer with unref()?

**Answer:** So the process never hangs forever, without keeping the event loop alive on its own

Keep it below the platform's kill timeout.

**Q156.** In what order should resources close?

**Answer:** In order, such as DB pools, Redis, message consumers, then telemetry exporters (to flush buffered spans and logs)

The example uses a db.end() step.

### [10 · Capstone — A Production-Ready Service](level-4/10-capstone.md)

**Q157.** Which key constraint does the Bookings capstone highlight?

**Answer:** Two overlapping bookings for the same resource must be impossible, even under concurrent requests

Constraints in the database enforce invariants.

**Q158.** When should the confirmation email job be enqueued?

**Answer:** When a booking is created, in the same transaction

A reminder job runs 30 minutes before the start.

**Q159.** How should the utilization report be computed?

**Answer:** Without blocking the event loop, whatever the date range

It draws on worker threads and streaming ideas from Level 3.

**Q160.** What shape should the domain and use cases have?

**Answer:** Independent of Express and the DB, with a composition root

The non-functional table also requires TypeScript strict, Zod at boundaries and one error format.

## Mixed questions across all levels

**Q161.** What is printed?

```javascript
import { EventEmitter } from 'node:events'
const e = new EventEmitter()
e.on('x', () => console.log('A'))
e.once('x', () => console.log('B'))
console.log('start')
console.log(e.emit('x'))
console.log(e.emit('x'))
console.log(e.emit('nobody'), e.listenerCount('x'))
```

**Answer:** start, A, B, true, A, true, false 1 (one per line)

emit is synchronous, so listeners run before emit returns true. The once listener is removed after the first call. emit returns false with no listeners, and one listener remains.

**Q162.** What does this print?

```javascript
console.log(Buffer.from('héllo €').length, 'héllo €'.length, Buffer.byteLength('€'))
```

**Answer:** 10 7 3

In UTF-8, é takes 2 bytes and € takes 3, so the string has 7 characters but 10 bytes.

**Q163.** What is printed?

```javascript
const b = Buffer.from('abcdef')
const v = b.subarray(0, 3); v[0] = 0x5a
console.log(b.toString())
const c = Buffer.from(b); c[0] = 0x71
console.log(b.toString(), c.toString())
```

**Answer:** Zbcdef, then Zbcdef qbcdef

subarray shares memory with the original, but Buffer.from(buffer) makes an independent copy.

**Q164.** What does this print?

```javascript
process.env.N = 5
console.log(typeof process.env.N, process.env.N, process.env.NOPE)
```

**Answer:** string 5 undefined

Values in process.env are always strings (or undefined when missing).

**Q165.** In what order are these lines printed?

```javascript
async function f() { console.log('f1'); await null; console.log('f2') }
f()
Promise.resolve().then(() => console.log('p1'))
console.log('sync')
```

**Answer:** f1, sync, f2, p1

f runs synchronously up to its first await; the continuation is queued as a microtask before the .then callback, so f2 prints before p1.

**Q166.** What is printed?

```javascript
const r = await Promise.allSettled([Promise.resolve(1), Promise.reject(new Error('no')), 3])
console.log(r.map(x => x.status).join(','))
```

**Answer:** fulfilled,rejected,fulfilled

allSettled never rejects: it reports each outcome. Promise.all would reject on the first failure.

**Q167.** What does this print?

```javascript
console.log(path.join('/srv/app', 'uploads', '../logs', 'app.log'), path.basename('/a/b/c.txt', '.txt'), path.extname('x.tar.gz'))
```

**Answer:** /srv/app/logs/app.log c .gz

path.join normalises '..', basename strips the given extension, and extname returns only the last extension.

**Q168.** What does this print?

```javascript
const u = new URL('/notes?q=Tea&q=Milk&limit=5', 'http://localhost')
console.log(u.pathname, u.searchParams.get('q'), u.searchParams.getAll('q').length, Number(u.searchParams.get('limit')) + 1)
```

**Answer:** /notes Tea 2 6

get returns the first value, getAll returns all of them, and query values are strings, so limit needs Number().

**Q169.** Which of these does the range ^5.1.0 accept, and which does ^0.4.2 accept? (5.9.1, 6.0.0 against ^5.1.0; 0.4.9, 0.5.0 against ^0.4.2)

**Answer:** ^5.1.0: 5.9.1 yes, 6.0.0 no. ^0.4.2: 0.4.9 yes, 0.5.0 no

Caret allows changes that don't bump the leftmost non-zero part, so for 0.x versions a minor bump may break.

**Q170.** What does this pipeline produce?

```javascript
const out = []
await pipeline(
  Readable.from(['a', 'b', 'c']),
  new Transform({ transform(ch, enc, cb) { cb(null, ch.toString().toUpperCase()) } }),
  async function (src) { for await (const x of src) out.push(x.toString()) },
)
console.log(out.join(''))
```

**Answer:** ABC

Transform maps each chunk to upper case; pipeline wires the streams and handles backpressure and errors.

**Q171.** A server hashes passwords with scrypt and a random salt per password. For the same password, are two stored hashes equal?

**Answer:** No: a different random salt gives a different hash, so compare with the stored salt and timingSafeEqual

The same password and the same salt gives equal output (verified by running), but a fresh salt does not.

**Q172.** What delays does exponential backoff with base 100 ms give for four retries?

**Answer:** 100, 200, 400, 800 ms (plus jitter in production)

base × 2^i. Add jitter so many clients don't retry at the same instant.

**Q173.** A circuit breaker has threshold 3. The calls fail, fail, fail, then succeed, succeed. What are the results?
(breaker returns 'open' while failures ≥ threshold)

**Answer:** fail, fail, fail, open, open

After three consecutive failures it fails fast until the cool-down, even though the dependency may have recovered (the simulated breaker has no cool-down here).

**Q174.** Why does a 500 ms CPU-bound loop make a 10 ms timer fire at about 500 ms?

**Answer:** Node runs JavaScript on one main thread, so the event loop can only check timers when the current code finishes

Move CPU-heavy work to worker_threads.

**Q175.** Inside an fs callback, which runs first: setImmediate or setTimeout(fn, 0)?

**Answer:** setImmediate, because the check phase follows the poll phase

Verified by running: 'immediate inside I/O' printed before 'timeout inside I/O'.

**Q176.** What should a service do on SIGTERM?

**Answer:** Report not ready, stop accepting connections, finish in-flight requests, close resources, then exit before SIGKILL

SIGKILL can't be caught, so use a hard deadline timer.

**Q177.** Why should file uploads be sniffed by bytes and renamed on disk?

**Answer:** The client's mimetype, extension and filename can lie or contain ../ paths; use magic numbers and a generated name

Also set size limits.

**Q178.** How does Express recognise an error-handling middleware, and where is it registered?

**Answer:** It takes four parameters (err, req, res, next) and is registered last

Express 5 also forwards rejected promises from async handlers to it.

**Q179.** A user asks for GET /tasks/17 owned by someone else. What should the API return, and why?

**Answer:** 404, so it doesn't reveal that the task exists, with the query scoped by owner

Scoping every query by the owner also prevents broken access control.

**Q180.** You run three cluster workers and keep a Map as a rate-limit counter in each. What goes wrong?

**Answer:** Each worker has its own heap, so counters aren't shared; use a shared store such as Redis

Clustering doesn't give shared memory.

## Practical coding questions

### Retry with exponential backoff

Write an async function retry(fn, { attempts, baseMs }) that calls fn(i). If it throws, wait baseMs * 2**i ms (not after the last attempt) and try again. After the last attempt rethrow the last error. Track the waits and print them for a function that fails twice and then succeeds.

Starter code:

```
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

Expected output:

```text
finally 3 10,20
```

??? success "Show a solution"

    ```
    import { setTimeout as sleep } from 'node:timers/promises'

    export async function retry(fn, { attempts = 3, baseMs = 10, onWait = () => {} } = {}) {
      let lastErr
      for (let i = 0; i < attempts; i++) {
        try {
          return await fn(i)
        } catch (err) {
          lastErr = err
          if (i < attempts - 1) {
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
    const result = await retry(async () => {
      if (++calls < 3) throw new Error('flaky')
      return 'finally'
    }, { attempts: 4, baseMs: 10, onWait: (ms) => waits.push(ms) })
    console.log(result, calls, waits.join(','))
    ```

    Waits grow as base × 2^i, and the last attempt doesn't wait. Verified by running the solution in Node: output matches.

### Parse access-log lines

Write parseLine(line) that matches the common log format and returns { method, path, status, bytes } or null for invalid lines. Then count requests by status class (2xx, 4xx…) for the sample lines.

Starter code:

```
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

Expected output:

```text
{"2xx":1,"4xx":2} invalid: 1
```

??? success "Show a solution"

    ```
    const LINE = /^(\S+) \S+ \S+ \[([^\]]+)\] "(\S+) (\S+) [^"]*" (\d{3}) (\d+|-)/

    export function parseLine(line) {
      const m = LINE.exec(line)
      if (!m) return null
      return { method: m[3], path: m[4], status: Number(m[5]), bytes: m[6] === '-' ? 0 : Number(m[6]) }
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

    Keeping the parser a pure function makes it easy to test. A '-' byte count becomes 0. Verified by running the solution in Node: output matches.

---

More free practice: the [Node.js Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
