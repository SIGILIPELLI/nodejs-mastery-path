---
description: "Test Node services with the built-in node:test runner and Supertest: unit tests with mocks, HTTP integration tests against a real database, and coverage."
---

# 08 · Testing with node:test & Supertest

An API without tests is an API you are afraid to change. Node now ships a capable test
runner in the box — `node:test` with `node:assert` — so a new project needs no test
framework dependency at all. For HTTP-level tests you add one small library,
**Supertest**, which sends requests to an Express app without opening a real port.

If your team already uses **Vitest** or **Jest**, everything in this lesson maps across
directly (`describe`/`test`/`expect`, and Supertest works the same). Vitest in particular
is a common choice for TypeScript projects and for sharing a runner with a Vite frontend.

## The shape of a test suite

| Kind | Tests | Speed | Example |
|------|-------|-------|---------|
| Unit | one function, no I/O | microseconds | `parseLine`, `retry`, a Zod schema |
| Integration | your app + real dependencies (DB) through HTTP | milliseconds | `POST /tasks` then `GET /tasks/:id` |
| End-to-end | deployed system from outside | seconds | smoke test after deploy |

For an API, integration tests through HTTP give the best confidence per test: they cover
routing, validation, middleware, SQL, and serialization together. Keep unit tests for
logic with many edge cases.

## Unit tests with node:test and mocks

```js title="src/retry.js"
import { setTimeout as sleep } from 'node:timers/promises';

// Retry an async function with exponential backoff
export async function retry(fn, { attempts = 3, baseMs = 100 } = {}) {
  let lastErr;
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn(i);
    } catch (err) {
      lastErr = err;
      if (i < attempts - 1) await sleep(baseMs * 2 ** i);
    }
  }
  throw lastErr;
}
```

```js title="test/retry.test.js"
import { test, mock } from 'node:test';
import assert from 'node:assert/strict';
import { retry } from './retry.js';

test('returns the first successful result', async () => {
  const fn = mock.fn(async () => 'ok');
  assert.equal(await retry(fn), 'ok');
  assert.equal(fn.mock.callCount(), 1);
});

test('retries until success', async () => {
  let calls = 0;
  const fn = mock.fn(async () => {
    if (++calls < 3) throw new Error('flaky');
    return 'finally';
  });
  assert.equal(await retry(fn, { baseMs: 1 }), 'finally');
  assert.equal(fn.mock.callCount(), 3);
});

test('gives up after the last attempt and rethrows', async () => {
  const fn = mock.fn(async () => { throw new Error('down'); });
  await assert.rejects(retry(fn, { attempts: 2, baseMs: 1 }), /down/);
  assert.equal(fn.mock.callCount(), 2);
});
```

```text
$ node --test --experimental-test-coverage 'test/*.test.js'
✔ returns the first successful result (1.6ms)
✔ retries until success (4.8ms)
✔ gives up after the last attempt and rethrows (2.1ms)
ℹ tests 3
ℹ pass 3
ℹ fail 0
ℹ file      | line % | branch % | funcs % | uncovered lines
ℹ  retry.js | 100.00 |   100.00 |  100.00 |
```

What's being used:

- `mock.fn(impl)` wraps a function and records calls (`fn.mock.callCount()`,
  `fn.mock.calls[0].arguments`).
- `assert/strict` makes `equal` use `===` and `deepEqual` strict deep equality.
- `assert.rejects(promise, /pattern/)` for async failures.
- Passing a tiny `baseMs` keeps the backoff test fast. For code that uses real timers
  with long delays, `mock.timers.enable({ apis: ['setTimeout'] })` lets you advance time
  with `mock.timers.tick(ms)`.

Useful runner flags: `--watch` (re-run on change), `--test-name-pattern="retries"`,
`--test-only` with `test.only(...)`, and `--test-concurrency`. File arguments are paths
or glob patterns; with no arguments, `node --test` searches for common test-file names.

## Integration tests with Supertest

The app factory from lesson 01 is what makes this pleasant: tests build an app wired to
a test database and a silent logger.

```js title="test/api.test.js (from the Level 2 project)"
import { describe, test, before, after } from 'node:test';
import assert from 'node:assert/strict';
import request from 'supertest';
import pino from 'pino';
import { createDb, migrate } from '../src/db.js';
import { createApp } from '../src/app.js';
import { loadConfig } from '../src/config.js';
import { pglitePool } from './pglite-pool.js';

const config = loadConfig({
  NODE_ENV: 'test',
  DATABASE_URL: 'postgres://unused/test',
  JWT_SECRET: 'test-secret-that-is-at-least-32-characters',
});

let db, app;
before(async () => {
  db = createDb({ pool: pglitePool() });   // in-process Postgres for tests
  await migrate(db);
  app = createApp({ db, config, logger: pino({ level: 'silent' }) });
});
after(() => db.destroy());

async function registerAndLogin(email) {
  const res = await request(app).post('/auth/register')
    .send({ email, password: 'correct horse battery' }).expect(201);
  return res.body.token;
}

describe('auth', () => {
  test('register, then login with the same credentials', async () => {
    await registerAndLogin('Ada@Example.com');
    const res = await request(app).post('/auth/login')
      .send({ email: 'ada@example.com', password: 'correct horse battery' }).expect(200);
    assert.match(res.body.token, /^[\w-]+\.[\w-]+\.[\w-]+$/);
  });

  test('duplicate email is 409', async () => {
    await request(app).post('/auth/register')
      .send({ email: 'ada@example.com', password: 'correct horse battery' }).expect(409);
  });

  test('wrong password is 401 with a generic message', async () => {
    const res = await request(app).post('/auth/login')
      .send({ email: 'ada@example.com', password: 'wrong password!!' }).expect(401);
    assert.equal(res.body.error, 'invalid email or password');
  });

  test('short password is rejected with field details', async () => {
    const res = await request(app).post('/auth/register')
      .send({ email: 'bob@example.com', password: 'short' }).expect(400);
    assert.equal(res.body.details[0].path, 'body.password');
  });
});

describe('tasks', () => {
  let token;
  before(async () => { token = await registerAndLogin('tasks@example.com'); });
  const auth = () => ({ Authorization: `Bearer ${token}` });

  test('requires a token', async () => {
    await request(app).get('/tasks').expect(401);
  });

  test('create, read, update, delete', async () => {
    const created = await request(app).post('/tasks').set(auth())
      .send({ title: '  write tests  ' }).expect(201);
    assert.equal(created.body.title, 'write tests');
    assert.equal(created.headers.location, `/tasks/${created.body.id}`);

    const patched = await request(app).patch(`/tasks/${created.body.id}`).set(auth())
      .send({ done: true }).expect(200);
    assert.equal(patched.body.done, true);

    await request(app).delete(`/tasks/${created.body.id}`).set(auth()).expect(204);
    await request(app).get(`/tasks/${created.body.id}`).set(auth()).expect(404);
  });

  test('cursor pagination', async () => {
    for (const title of ['a', 'b', 'c']) {
      await request(app).post('/tasks').set(auth()).send({ title }).expect(201);
    }
    const page1 = await request(app).get('/tasks?limit=2').set(auth()).expect(200);
    assert.deepEqual(page1.body.data.map(t => t.title), ['a', 'b']);
    const page2 = await request(app).get(`/tasks?limit=2&cursor=${page1.body.nextCursor}`).set(auth()).expect(200);
    assert.deepEqual(page2.body.data.map(t => t.title), ['c']);
    assert.equal(page2.body.nextCursor, null);
  });

  test("users cannot see each other's tasks", async () => {
    const mine = await request(app).post('/tasks').set(auth()).send({ title: 'private' }).expect(201);
    const other = await registerAndLogin('mallory@example.com');
    await request(app).get(`/tasks/${mine.body.id}`)
      .set({ Authorization: `Bearer ${other}` }).expect(404);
  });

  test('malformed JSON is a 400, not a 500', async () => {
    await request(app).post('/tasks').set(auth())
      .set('content-type', 'application/json').send('{"title":').expect(400);
  });
});
```

```text
▶ auth
  ✔ register, then login with the same credentials
  ✔ duplicate email is 409
  ✔ wrong password is 401 with a generic message
  ✔ short password is rejected with field details
▶ tasks
  ✔ requires a token
  ✔ create, read, update, delete
  ✔ cursor pagination
  ✔ users cannot see each other's tasks
  ✔ malformed JSON is a 400, not a 500
ℹ tests 9
ℹ pass 9
ℹ fail 0
```

Notes on the choices:

- **A real Postgres engine, not a mock.** Mocking the database would test your mocks. Here
  `pglitePool()` is a ~15-line adapter that lets Kysely's Postgres dialect talk to
  **PGlite** (Postgres compiled to WebAssembly, `npm i -D @electric-sql/pglite`), so tests
  exercise real SQL, constraints, and error codes (`23505`) with no server to install. For
  full fidelity — extensions, exact driver type parsing — run against a real Postgres in
  Docker or a CI service container; the tests don't change, only the pool passed to
  `createDb`.
- **Behavior, not implementation.** Tests assert status codes and response bodies, not
  which functions were called. You can refactor internals freely.
- **Security properties get tests** — the "users cannot see each other's tasks" test
  guards against the most common API vulnerability (broken object-level authorization).
- **Supertest's `request(app)`** calls `app.listen(0)` on an ephemeral port internally
  and closes it after the request, so tests never collide on ports.

The adapter, for reference:

```js title="test/pglite-pool.js"
// Test-only adapter: lets Kysely's PostgresDialect talk to in-process PGlite
import { PGlite } from '@electric-sql/pglite';
export function pglitePool() {
  const pg = new PGlite();
  const client = {
    async query(sql, params) {
      const r = await pg.query(sql, params);
      const command = sql.trim().split(/\s+/)[0].toUpperCase();
      return { rows: r.rows, rowCount: r.affectedRows ?? r.rows.length, command };
    },
    release() {},
  };
  return { connect: async () => client, end: async () => pg.close() };
}
```

## How It Actually Works

`node:test` builds a tree of tests as your files execute: each `test()`/`describe()` call
registers a node, and hooks (`before`, `beforeEach`, `after`) attach to their enclosing
suite. When `node --test` runs, it spawns **one child process per test file** by
default, so files are isolated from each other's module state and can run in parallel;
tests *within* a file run sequentially unless you opt in. Each child reports results to
the parent over a stream in a structured format, and the parent renders them with a
reporter (`spec` in a terminal, `tap` or `junit` for CI via `--test-reporter`).

A test passes if its function returns (or its promise resolves) without throwing. That's
why forgetting `await` before an async assertion produces false passes: the assertion
rejects after the test has already been marked successful.

Coverage uses V8's built-in **precise coverage**: V8 counts executed blocks and
functions as your code runs, and Node maps those counts back to source lines and
branches at the end — no code instrumentation step.

Supertest wraps `superagent`. Given an Express app (a request-handler function), it
creates an `http.Server`, listens on port 0 so the OS assigns a free port, performs the
request with a real HTTP client over loopback, and closes the server. Your app goes
through exactly the same HTTP parsing path as in production.

## Common mistakes

- **Missing `await`** on `request(...)` or `assert.rejects` → tests that can't fail.
- **Tests depending on order** or shared mutable data between files. Give each test its
  own users/rows (the project registers a fresh user per suite).
- **Mocking what you own** (your repository layer) in HTTP tests, then shipping broken SQL.
- **Asserting on log output or exact error strings** that change often; assert on status
  and structured fields.
- **Slow suites from real sleeps.** Inject delays or use mock timers.
- **Never running tests in CI.** Add `npm test` to your pipeline on every push.

## Exercise

1. Write unit tests for the `validate` middleware by calling it directly with fake
   `req`, `res`, and a `mock.fn()` as `next`. Assert that `next` receives an `HttpError`
   with status 400 on bad input.
2. Add an integration test that a JWT signed with the wrong secret gets 401, and one that
   an expired token gets 401 (sign with `expiresIn: -1`).
3. Add a test for `PATCH /tasks/:id` with `{}` asserting the refinement message appears in
   `details`.
4. Enable coverage with a threshold: `node --test --experimental-test-coverage
   --test-coverage-lines=90`. Find the uncovered lines and decide whether they deserve
   tests.
