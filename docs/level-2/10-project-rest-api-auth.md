---
description: "Project: a complete Express 5 tasks API with PostgreSQL via Kysely, scrypt passwords, JWT auth, Zod validation, Pino logs, and a passing Supertest suite."
---

# 10 · Project — A REST API with Auth

Time to assemble Level 2 into one service you could actually deploy: a multi-user
**tasks API**. Users register and log in; each user can create, list (with filtering and
cursor pagination), update, and delete *their own* tasks. Every piece comes from a
lesson in this level, and the whole thing is covered by an integration test suite that
runs against a real Postgres engine.

## Requirements

- `POST /auth/register`, `POST /auth/login` → JWT access token (15 minutes).
- `GET /tasks?done=&limit=&cursor=`, `POST /tasks`, `GET/PATCH/DELETE /tasks/:id`,
  all requiring `Authorization: Bearer <token>`.
- Users can never read or modify another user's tasks (404, not 403).
- Validation errors → 400 with per-field `details`; malformed JSON → 400; duplicates →
  409; unknown routes → 404 JSON; bugs → 500 without internals.
- Structured request logs with a propagated `x-request-id`.
- Configuration validated at startup; graceful shutdown on `SIGTERM`.

## Layout

```text
tasks-api/
  package.json
  .env.example
  src/
    config.js          # Zod-validated env (lesson 04)
    db.js              # Kysely + pg pool, schema bootstrap (lesson 06)
    errors.js          # HttpError + error middleware (lesson 05)
    validate.js        # validation middleware (lesson 04)
    auth/password.js   # scrypt hashing (lesson 07)
    auth/tokens.js     # JWT sign + requireAuth (lesson 07)
    routes/auth.js
    routes/tasks.js    # REST + cursor pagination (lesson 03)
    app.js             # createApp factory (lessons 01-02, 09)
    server.js          # entry point: config, db, listen, shutdown
  test/
    api.test.js        # Supertest integration tests (lesson 08)
    pglite-pool.js     # test-only in-process Postgres adapter
```

## package.json

```json title="package.json"
{
  "name": "tasks-api",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "start": "node src/server.js",
    "dev": "node --watch --env-file=.env src/server.js",
    "test": "node --test 'test/*.test.js'"
  },
  "dependencies": {
    "express": "^5.1.0",
    "jsonwebtoken": "^9.0.2",
    "kysely": "^0.29.0",
    "pg": "^8.16.0",
    "pino": "^10.0.0",
    "pino-http": "^11.0.0",
    "zod": "^4.0.0"
  },
  "devDependencies": {
    "@electric-sql/pglite": "^0.5.0",
    "supertest": "^7.1.0"
  }
}
```

```text title=".env.example"
PORT=3000
DATABASE_URL=postgres://tasks:devpass@localhost:5432/tasks
JWT_SECRET=replace-with-at-least-32-random-characters
LOG_LEVEL=debug
```

## Configuration and database

```js title="src/config.js"
import { z } from 'zod';

const schema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.url(),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace', 'silent']).default('info'),
});

export function loadConfig(env = process.env) {
  const parsed = schema.safeParse(env);
  if (!parsed.success) {
    const problems = parsed.error.issues.map(i => `  ${i.path.join('.')}: ${i.message}`).join('\n');
    throw new Error(`Invalid configuration:\n${problems}`);
  }
  return Object.freeze(parsed.data);
}
```

```js title="src/db.js"
import { Kysely, PostgresDialect, sql } from 'kysely';
import pg from 'pg';

export function createDb({ connectionString, pool } = {}) {
  return new Kysely({
    dialect: new PostgresDialect({ pool: pool ?? new pg.Pool({ connectionString, max: 10 }) }),
  });
}

export async function migrate(db) {
  await db.schema.createTable('users').ifNotExists()
    .addColumn('id', 'serial', c => c.primaryKey())
    .addColumn('email', 'text', c => c.notNull().unique())
    .addColumn('password_hash', 'text', c => c.notNull())
    .addColumn('created_at', 'timestamptz', c => c.notNull().defaultTo(sql`now()`))
    .execute();
  await db.schema.createTable('tasks').ifNotExists()
    .addColumn('id', 'serial', c => c.primaryKey())
    .addColumn('user_id', 'integer', c => c.notNull().references('users.id').onDelete('cascade'))
    .addColumn('title', 'text', c => c.notNull())
    .addColumn('done', 'boolean', c => c.notNull().defaultTo(false))
    .addColumn('created_at', 'timestamptz', c => c.notNull().defaultTo(sql`now()`))
    .execute();
  await db.schema.createIndex('tasks_user_id_idx').ifNotExists().on('tasks').column('user_id').execute();
}
```

`migrate()` here is a simple idempotent bootstrap (`ifNotExists`) to keep the project
compact. Once the schema starts evolving, switch to versioned migrations with Kysely's
`Migrator` as shown in lesson 06.

## Errors and validation

```js title="src/errors.js"
export class HttpError extends Error {
  constructor(status, message, details) {
    super(message);
    this.name = 'HttpError';
    this.status = status;
    this.details = details;
  }
}

export const badRequest = (msg, details) => new HttpError(400, msg, details);
export const unauthorized = (msg = 'authentication required') => new HttpError(401, msg);
export const notFound = (msg = 'not found') => new HttpError(404, msg);
export const conflict = (msg) => new HttpError(409, msg);

// Express 5: registered last, with four parameters
export function errorHandler(err, req, res, next) {
  if (res.headersSent) return next(err);

  // body-parser errors (malformed JSON, too large) carry a status and `expose`
  if (err.type === 'entity.parse.failed') err = badRequest('malformed JSON body');
  const status = err.status ?? 500;

  if (status >= 500) req.log.error({ err }, 'unhandled error');
  else req.log.warn({ status, msg: err.message }, 'request failed');

  res.status(status).json({
    error: status >= 500 ? 'internal server error' : err.message,
    ...(err.details && { details: err.details }),
  });
}
```

```js title="src/validate.js"
import { badRequest } from './errors.js';

// validate({ body: schema, query: schema, params: schema })
export function validate(schemas) {
  return (req, res, next) => {
    for (const [part, schema] of Object.entries(schemas)) {
      const result = schema.safeParse(req[part]);
      if (!result.success) {
        const details = result.error.issues.map(i => ({ path: [part, ...i.path].join('.'), message: i.message }));
        return next(badRequest('validation failed', details));
      }
      // Express 5 makes req.query a getter, so store parsed values separately
      req.valid ??= {};
      req.valid[part] = result.data;
    }
    next();
  };
}
```

## Authentication

```js title="src/auth/password.js"
import { scrypt, randomBytes, timingSafeEqual } from 'node:crypto';
import { promisify } from 'node:util';

const scryptAsync = promisify(scrypt);
const KEYLEN = 64;
const PARAMS = { N: 16384, r: 8, p: 1 }; // cost parameters, stored with each hash

export async function hashPassword(password) {
  const salt = randomBytes(16);
  const key = await scryptAsync(password, salt, KEYLEN, PARAMS);
  return `scrypt$${PARAMS.N}$${PARAMS.r}$${PARAMS.p}$${salt.toString('base64')}$${key.toString('base64')}`;
}

export async function verifyPassword(password, stored) {
  const [algo, N, r, p, saltB64, keyB64] = stored.split('$');
  if (algo !== 'scrypt') return false;
  const expected = Buffer.from(keyB64, 'base64');
  const actual = await scryptAsync(password, Buffer.from(saltB64, 'base64'), expected.length,
    { N: Number(N), r: Number(r), p: Number(p) });
  return timingSafeEqual(actual, expected);
}
```

```js title="src/auth/tokens.js"
import jwt from 'jsonwebtoken';
import { unauthorized } from '../errors.js';

export function signToken(user, secret) {
  return jwt.sign({ sub: String(user.id), email: user.email }, secret, {
    algorithm: 'HS256',
    expiresIn: '15m',
  });
}

export function requireAuth(secret) {
  return (req, res, next) => {
    const header = req.get('authorization') ?? '';
    const [scheme, token] = header.split(' ');
    if (scheme !== 'Bearer' || !token) return next(unauthorized());
    try {
      const payload = jwt.verify(token, secret, { algorithms: ['HS256'] });
      req.user = { id: Number(payload.sub), email: payload.email };
      next();
    } catch {
      next(unauthorized('invalid or expired token'));
    }
  };
}
```

```js title="src/routes/auth.js"
import { Router } from 'express';
import { z } from 'zod';
import { validate } from '../validate.js';
import { conflict, unauthorized } from '../errors.js';
import { hashPassword, verifyPassword } from '../auth/password.js';
import { signToken } from '../auth/tokens.js';

const credentials = z.object({
  email: z.email().transform(e => e.toLowerCase()),
  password: z.string().min(12, 'password must be at least 12 characters').max(200),
});

export function authRouter({ db, config }) {
  const router = Router();

  router.post('/register', validate({ body: credentials }), async (req, res) => {
    const { email, password } = req.valid.body;
    const password_hash = await hashPassword(password);
    try {
      const user = await db.insertInto('users').values({ email, password_hash })
        .returning(['id', 'email']).executeTakeFirstOrThrow();
      res.status(201).json({ user, token: signToken(user, config.JWT_SECRET) });
    } catch (err) {
      if (err.code === '23505') throw conflict('email already registered');
      throw err;
    }
  });

  router.post('/login', validate({ body: credentials }), async (req, res) => {
    const { email, password } = req.valid.body;
    const user = await db.selectFrom('users').select(['id', 'email', 'password_hash'])
      .where('email', '=', email).executeTakeFirst();
    // Same error for unknown email and wrong password: don't reveal which accounts exist
    if (!user || !(await verifyPassword(password, user.password_hash))) {
      throw unauthorized('invalid email or password');
    }
    res.json({ token: signToken(user, config.JWT_SECRET) });
  });

  return router;
}
```

## Tasks

```js title="src/routes/tasks.js"
import { Router } from 'express';
import { z } from 'zod';
import { validate } from '../validate.js';
import { notFound } from '../errors.js';

const idParams = z.object({ id: z.coerce.number().int().positive() });
const createBody = z.object({ title: z.string().trim().min(1).max(200) });
const updateBody = z.object({
  title: z.string().trim().min(1).max(200).optional(),
  done: z.boolean().optional(),
}).refine(b => Object.keys(b).length > 0, 'provide at least one field');
const listQuery = z.object({
  done: z.enum(['true', 'false']).transform(v => v === 'true').optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.coerce.number().int().positive().optional(),
});

const columns = ['id', 'title', 'done', 'created_at'];

export function tasksRouter({ db }) {
  const router = Router();

  router.get('/', validate({ query: listQuery }), async (req, res) => {
    const { done, limit, cursor } = req.valid.query;
    let q = db.selectFrom('tasks').select(columns)
      .where('user_id', '=', req.user.id)
      .orderBy('id').limit(limit + 1);
    if (done !== undefined) q = q.where('done', '=', done);
    if (cursor) q = q.where('id', '>', cursor);
    const rows = await q.execute();
    const hasMore = rows.length > limit;
    const data = rows.slice(0, limit);
    res.json({ data, nextCursor: hasMore ? data.at(-1).id : null });
  });

  router.post('/', validate({ body: createBody }), async (req, res) => {
    const task = await db.insertInto('tasks')
      .values({ user_id: req.user.id, title: req.valid.body.title })
      .returning(columns).executeTakeFirstOrThrow();
    res.status(201).location(`${req.baseUrl}/${task.id}`).json(task);
  });

  router.get('/:id', validate({ params: idParams }), async (req, res) => {
    const task = await db.selectFrom('tasks').select(columns)
      .where('id', '=', req.valid.params.id).where('user_id', '=', req.user.id)
      .executeTakeFirst();
    if (!task) throw notFound('task not found');
    res.json(task);
  });

  router.patch('/:id', validate({ params: idParams, body: updateBody }), async (req, res) => {
    const task = await db.updateTable('tasks').set(req.valid.body)
      .where('id', '=', req.valid.params.id).where('user_id', '=', req.user.id)
      .returning(columns).executeTakeFirst();
    if (!task) throw notFound('task not found');
    res.json(task);
  });

  router.delete('/:id', validate({ params: idParams }), async (req, res) => {
    const result = await db.deleteFrom('tasks')
      .where('id', '=', req.valid.params.id).where('user_id', '=', req.user.id)
      .executeTakeFirst();
    if (result.numDeletedRows === 0n) throw notFound('task not found');
    res.status(204).end();
  });

  return router;
}
```

Every query includes `where('user_id', '=', req.user.id)`. Authorization is enforced in
the data access itself, so there is no code path that loads a task and forgets to check
its owner.

## Wiring

```js title="src/app.js"
import express from 'express';
import { pinoHttp } from 'pino-http';
import { randomUUID } from 'node:crypto';
import { authRouter } from './routes/auth.js';
import { tasksRouter } from './routes/tasks.js';
import { requireAuth } from './auth/tokens.js';
import { errorHandler, notFound } from './errors.js';

export function createApp({ db, config, logger }) {
  const app = express();
  app.disable('x-powered-by');

  app.use(pinoHttp({
    logger,
    genReqId: (req, res) => {
      const id = req.get('x-request-id') ?? randomUUID();
      res.setHeader('x-request-id', id);
      return id;
    },
  }));
  app.use(express.json({ limit: '100kb' }));

  app.get('/health', (req, res) => res.json({ status: 'ok' }));
  app.use('/auth', authRouter({ db, config }));
  app.use('/tasks', requireAuth(config.JWT_SECRET), tasksRouter({ db }));

  app.use((req, res, next) => next(notFound(`no route for ${req.method} ${req.path}`)));
  app.use(errorHandler);
  return app;
}
```

```js title="src/server.js"
import pino from 'pino';
import { loadConfig } from './config.js';
import { createDb, migrate } from './db.js';
import { createApp } from './app.js';

const config = loadConfig();
const logger = pino({ level: config.LOG_LEVEL });
const db = createDb({ connectionString: config.DATABASE_URL });
await migrate(db);

const server = createApp({ db, config, logger }).listen(config.PORT, () => {
  logger.info({ port: config.PORT }, 'server listening');
});

async function shutdown(signal) {
  logger.info({ signal }, 'shutting down');
  server.close(async () => {
    await db.destroy();
    process.exit(0);
  });
  setTimeout(() => process.exit(1), 10_000).unref();
}
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

## Tests

```js title="test/api.test.js"
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
$ npm test
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

## Trying it by hand

With a local Postgres running and `.env` filled in, `npm run dev` starts the server.
(For the session below, the same `createApp` was started on port 3100 with the
in-process PGlite pool instead of a Postgres server.)

```text
$ curl -X POST localhost:3100/auth/register -H 'content-type: application/json' \
    -d '{"email":"ada@example.com","password":"correct horse battery"}'
{"user":{"id":1,"email":"ada@example.com"},"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...."}

$ curl -i -X POST localhost:3100/tasks -H "authorization: Bearer $TOKEN" \
    -H 'content-type: application/json' -d '{"title":"ship level 2"}'
HTTP/1.1 201 Created
x-request-id: fbd5e9c7-bfc5-4f94-8a6f-bffda9222838
Location: /tasks/1
{"id":1,"title":"ship level 2","done":false,"created_at":"2026-09-26T15:08:16.921Z"}

$ curl "localhost:3100/tasks?done=false" -H "authorization: Bearer $TOKEN"
{"data":[{"id":1,"title":"ship level 2","done":false,"created_at":"2026-09-26T15:08:16.921Z"}],"nextCursor":null}

$ curl -X PATCH localhost:3100/tasks/1 -H "authorization: Bearer $TOKEN" \
    -H 'content-type: application/json' -d '{}'
{"error":"validation failed","details":[{"path":"body","message":"provide at least one field"}]}

$ curl localhost:3100/tasks
{"error":"authentication required"}
```

## How It Actually Works

Follow one request — `PATCH /tasks/1` with a valid token — through the stack:

1. **node:http** parses the request line and headers (llhttp) and calls the Express app.
2. **pino-http** assigns the request id, attaches `req.log`, and registers a `'finish'`
   listener for the completion log line.
3. **express.json** streams the body, enforces the 100 KB limit, and parses JSON into
   `req.body` (or raises `entity.parse.failed`).
4. `/health` and `/auth` don't match, so the router reaches `app.use('/tasks', ...)`:
   **requireAuth** verifies the HMAC signature and expiry of the JWT and sets `req.user`.
5. The **tasks router** strips `/tasks` from the path, matches `PATCH /:id`, and runs
   **validate**, which coerces `id` to a number and parses the body.
6. The handler awaits **Kysely**, which compiles an `UPDATE ... WHERE id = $1 AND
   user_id = $2 RETURNING ...`, borrows a pooled connection, and sends it using the
   extended query protocol. The event loop serves other requests meanwhile.
7. The row comes back; `res.json` serializes it; the `'finish'` event fires and pino-http
   writes the completion line with status and duration.

If anything throws in steps 4–6, Express 5 routes the error to `errorHandler`, which maps
it to a status and logs with the same request id.

## Common mistakes in projects like this

- Checking ownership in some handlers and forgetting it in others. Put it in the query.
- Returning `password_hash` by selecting `*` from `users`. Select explicit columns.
- A test suite that shares one user across all tests, so order matters.
- A `JWT_SECRET` default in code "for development" that reaches production.
- No limit on list endpoints.

## Exercise

Extend the project; each item should come with tests.

1. **Projects:** add a `projects` table; tasks optionally belong to a project the user
   owns. Creating a task in someone else's project must fail with 404.
2. **Refresh tokens:** implement the refresh flow from lesson 07, including logout that
   revokes the refresh token.
3. **Rate limiting:** limit `/auth/login` to 5 attempts per minute per IP and email.
4. **Search:** `GET /tasks?q=milk` using Postgres `ILIKE`, with the search term passed as a
   parameter, and a test proving `%` and `_` in the search term are matched literally.
5. **Real Postgres in CI:** run the same tests against a Postgres service container by
   choosing the pool from an environment variable.
