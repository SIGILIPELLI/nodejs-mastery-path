---
title: "Node.js Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Node.js cheat sheet: 15 quick-reference tables from every level of the Node.js Mastery Path, each linked to its full lesson."
---

# Node.js Cheat Sheet

Quick-reference tables from **15 lessons** of the free Node.js Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [03 · Modules: ES Modules &amp; CommonJS](level-1/03-modules-esm-cjs.md)

*How Node decides which system a file uses*

| File | Treated as |
|------|-----------|
| `*.mjs` | always ESM |
| `*.cjs` | always CommonJS |
| `*.js` with nearest `package.json` containing `"type": "module"` | ESM |
| `*.js` with `"type": "commonjs"` or no `type` field | CommonJS (with a fallback, below) |

### [04 · npm &amp; package.json](level-1/04-npm-package-json.md)

*Semantic versioning and ranges*

| Range | Accepts | Notes |
|-------|---------|-------|
| `^5.1.0` | `>=5.1.0 <6.0.0` | npm's default when you `npm install` |
| `~5.1.0` | `>=5.1.0 <5.2.0` | patches only |
| `5.1.0` | exactly that | use `npm install --save-exact` |
| `^0.4.2` | `>=0.4.2 <0.5.0` | for `0.x`, caret is stricter: minor bumps may break |

### [05 · Files &amp; Paths: fs and path](level-1/05-fs-and-path.md)

*Three flavors of fs*

| Style | When to use |
|-------|------------|
| `fs/promises` + `await` | Default for application code and servers |
| `*Sync` | Startup code, CLIs, build scripts — where blocking is harmless |
| Callbacks | Older code; no advantage in new code |
| Streams (`createReadStream`) | Large files, or when you want to start processing before the whole file is in memory |

## Level 2 · Intermediate

### [01 · Express Fundamentals](level-2/01-express-fundamentals.md)

*The objects you work with*

| On `req` | Meaning |
|---------|---------|
| `req.params` | route parameters (`/books/:id` → `{ id: '42' }`), always strings |
| `req.query` | parsed query string (`?tag=a&tag=b` → `{ tag: ['a', 'b'] }`) |
| `req.body` | parsed body — only if a body-parsing middleware ran |
| `req.get('authorization')` | case-insensitive header lookup |
| `req.path`, `req.originalUrl`, `req.ip`, `req.method` | request metadata |

### [03 · REST API Design](level-2/03-rest-api-design.md)

*Resources and URLs*

| Operation | Method + path | Success status |
|-----------|---------------|----------------|
| List tasks | `GET /tasks` | 200 |
| Create a task | `POST /tasks` | 201 + `Location: /tasks/17` |
| Read one | `GET /tasks/17` | 200 |
| Partial update | `PATCH /tasks/17` | 200 (with body) or 204 |
| Replace | `PUT /tasks/17` | 200 or 204 |
| Delete | `DELETE /tasks/17` | 204 |
| Sub-collection | `GET /projects/3/tasks` | 200 |

### [06 · PostgreSQL with Kysely](level-2/06-databases-postgres.md)

| Tool | Style | Notes |
|------|-------|-------|
| `pg` alone | raw SQL strings with `$1` params | Maximum control, no abstraction |
| Knex | query builder | Long-established, JavaScript-first |
| Drizzle | SQL-like builder + schema in TS | Schema-as-code, TS-first |
| Prisma | ORM with its own schema file and generated client | High-level, strong tooling, less SQL control |
| TypeORM / Sequelize / MikroORM | class-based ORMs | Entity classes, unit-of-work patterns |

### [08 · Testing with node:test &amp; Supertest](level-2/08-testing.md)

*The shape of a test suite*

| Kind | Tests | Speed | Example |
|------|-------|-------|---------|
| Unit | one function, no I/O | microseconds | `parseLine`, `retry`, a Zod schema |
| Integration | your app + real dependencies (DB) through HTTP | milliseconds | `POST /tasks` then `GET /tasks/:id` |
| End-to-end | deployed system from outside | seconds | smoke test after deploy |

## Level 3 · Advanced

### [02 · Streams &amp; Backpressure](level-3/02-streams-backpressure.md)

*The four kinds*

| Type | You… | Examples |
|------|------|----------|
| Readable | read from it | `fs.createReadStream`, `req` on a server, `process.stdin` |
| Writable | write to it | `fs.createWriteStream`, `res` on a server, `process.stdout` |
| Duplex | both, independent sides | TCP `net.Socket`, WebSocket streams |
| Transform | both, output derived from input | `zlib.createGzip()`, `crypto.createHash()`, a CSV parser |

### [04 · Worker Threads &amp; Child Processes](level-3/04-worker-threads-child-processes.md)

*Child processes*

| Function | Shell? | Output | Use for |
|----------|--------|--------|---------|
| `execFile(cmd, args)` | no | buffered (limit: `maxBuffer`) | short commands with small output |
| `spawn(cmd, args)` | no (unless `shell: true`) | streams | long-running or large output |
| `exec('cmd string')` | **yes** | buffered | only with fully trusted, constant strings |
| `fork(modulePath)` | no | streams + IPC channel | another Node process you control |

### [09 · Security for Node Services](level-3/09-security.md)

*The OWASP categories, in Node terms*

| Risk | What it looks like in a Node API | Defense (lesson) |
|------|----------------------------------|------------------|
| Broken access control | `GET /tasks/:id` returns any user's task | Scope every query by owner (L2-10) |
| Injection | SQL built with template strings; `exec()` with user input | Parameterized queries (L2-06), `execFile` with arg arrays (L3-04) |
| Cryptographic failures | Fast hashes for passwords, secrets in code | scrypt/argon2 (L2-07), config from env (L1-09) |
| Insecure design | No rate limit on login, unbounded list endpoints | Rate limiting (below), pagination (L2-03) |
| Security misconfiguration | Stack traces in 500s, permissive CORS, missing headers | Error handler (L2-05), CORS allowlist (L2-02), helmet (below) |
| Vulnerable components | Outdated packages with known CVEs | `npm audit`, lockfiles, updates (below) |
| Auth failures | Long-lived JWTs, session fixation | Short tokens, `regenerate()` (L2-07) |
| Integrity failures | Compromised dependency, malicious install script | Lockfile integrity, `--ignore-scripts`, provenance (below) |
| Logging failures | No logs of auth failures; tokens in logs | Pino with redaction (L2-09) |
| SSRF | Server fetches a user-supplied URL | URL allowlists (below) |

### [10 · Project — A Real-Time Chat App](level-3/10-project-realtime-app.md)

*Features and the lesson each comes from*

| Feature | Lesson |
|---------|--------|
| Upgrade-time auth with a signed, expiring token and Origin check | 07 WebSockets, 03 Buffers (HMAC bytes) |
| Messages validated with a Zod discriminated union | L2-04 |
| Rooms, presence, typing, bounded history | 07 |
| Slow-consumer protection via `bufferedAmount` | 02 Streams & backpressure |
| Per-connection token-bucket rate limit | 09 Security |
| Heartbeats with ping/pong | 07 |
| Graceful shutdown with close code 1001 | preview of L4-09 |

## Level 4 · Master

### [02 · Diagnostics in Production](level-4/02-diagnostics.md)

*Quick diagnostic flags*

| Flag / variable | What it tells you |
|-----------------|-------------------|
| `--trace-warnings` | stack trace for every process warning (e.g. `MaxListenersExceededWarning`) |
| `--trace-uncaught` | where an uncaught exception was *thrown*, even for non-`Error` values |
| `--trace-deprecation` | stack traces for deprecation warnings |
| `--unhandled-rejections=strict` | make unhandled rejections fail fast everywhere |
| `NODE_DEBUG=http,net` | verbose internal debug logs from core modules |
| `--trace-gc` | a line per garbage collection with timing |
| `--heapsnapshot-signal=SIGUSR2` | write a heap snapshot on a signal |
| `--inspect=127.0.0.1:9229` | attach Chrome DevTools/VS Code |

### [05 · Queues &amp; Background Jobs](level-4/05-queues-background-jobs.md)

*Options in the Node ecosystem*

| Backend | Examples | Notes |
|---------|----------|-------|
| Redis | **BullMQ** | The most widely used Node job library: delays, retries, priorities, rate limits, repeatable jobs, dashboards |
| PostgreSQL | pg-boss, Graphile Worker, or hand-rolled | No extra infrastructure; jobs can be enqueued in the same transaction as your data |
| Message brokers | RabbitMQ, Amazon SQS, Kafka, NATS | For cross-service messaging and very high throughput (next lesson) |

### [09 · Graceful Shutdown &amp; Reliability](level-4/09-graceful-shutdown-reliability.md)

*Timeouts everywhere*

| Wait | Bound |
|------|-------|
| Outbound HTTP | `AbortSignal.timeout(ms)` on every `fetch` (lesson 06) |
| Database | pool acquire timeout, `statement_timeout` in Postgres |
| Redis | client command timeout |
| Incoming requests | `server.requestTimeout`, `server.headersTimeout` (L1-08) |
| Whole request | a per-request deadline middleware; pass the signal down |
| Shutdown | the force-exit timer above |

### [10 · Capstone — A Production-Ready Service](level-4/10-capstone.md)

*Non-functional requirements*

| Area | Requirement | Where you learned it |
|------|-------------|----------------------|
| Structure | Domain + use cases independent of Express and the DB; composition root | L4-04 |
| Types | TypeScript, `strict`, `tsc` in CI; Zod at every boundary | L4-03, L2-04 |
| Data | PostgreSQL via Kysely (or your justified alternative); versioned migrations; constraints enforce invariants | L2-06 |
| Errors | One error format; domain errors mapped at the edge; no internals leaked | L2-05 |
| Security | scrypt/argon2, rate-limited auth, helmet, ownership checks in queries, `npm audit` clean at high severity | L2-07, L3-09 |
| Tests | Use-case tests with fakes; HTTP integration tests against real Postgres (PGlite or container); a concurrency test for double booking | L2-08 |
| Observability | Pino logs with request/trace ids and redaction; OTel traces; RED + event-loop metrics | L2-09, L4-08 |
| Performance | A CPU profile of the report endpoint under load, and what you changed | L4-01 |
| Operations | Dockerfile (multi-stage, non-root), `/livez` + `/readyz`, graceful shutdown with deadline, config validated at startup | L4-07, L4-09, L1-09 |
| Workers | Separate worker process for jobs; idempotent handlers; dead-letter handling | L4-05 |

---

Go deeper with the full [Node.js Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
