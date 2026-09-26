---
description: "Use PostgreSQL from Node with the pg driver and the Kysely query builder: pools, parameterized queries, joins, transactions, migrations, and alternatives."
---

# 06 · PostgreSQL with Kysely

Almost every real API needs durable storage, and PostgreSQL is the default choice for
most new backends: relational, transactional, mature, and flexible (JSON columns,
full-text search, strong constraints). This lesson connects Node to Postgres through two
layers:

- **`pg`** (node-postgres) — the driver. It speaks Postgres's wire protocol and manages a
  **connection pool**.
- **Kysely** — a type-safe **query builder** on top of the driver. You write queries in
  JavaScript that map almost one-to-one onto SQL; Kysely compiles them to parameterized SQL
  strings.

We present Kysely because it keeps you close to SQL (so the SQL knowledge transfers)
while removing string concatenation. Other common options:

| Tool | Style | Notes |
|------|-------|-------|
| `pg` alone | raw SQL strings with `$1` params | Maximum control, no abstraction |
| Knex | query builder | Long-established, JavaScript-first |
| Drizzle | SQL-like builder + schema in TS | Schema-as-code, TS-first |
| Prisma | ORM with its own schema file and generated client | High-level, strong tooling, less SQL control |
| TypeORM / Sequelize / MikroORM | class-based ORMs | Entity classes, unit-of-work patterns |

```bash
npm install kysely pg
```

You need a PostgreSQL server to run this against. Install it locally or run it in a
container (`docker run -e POSTGRES_PASSWORD=dev -p 5432:5432 postgres`). The outputs
below were produced by running the same Kysely code against **PGlite**, a WebAssembly
build of Postgres that runs inside Node — handy for tests, and used for the Level 2
project's test suite.

## Connecting

```js title="src/db.js"
import { Kysely, PostgresDialect } from 'kysely';
import pg from 'pg';

export function createDb({ connectionString, pool } = {}) {
  return new Kysely({
    dialect: new PostgresDialect({ pool: pool ?? new pg.Pool({ connectionString, max: 10 }) }),
  });
}
```

Create **one** `Kysely` instance (one pool) per process at startup and pass it to your
routers. Call `db.destroy()` on shutdown to close connections.

## Schema and migrations

Tables are created by **migrations**: versioned scripts applied in order, recorded in a
table so each runs once. Kysely includes a `Migrator` that runs migration files exporting
`up` (and optionally `down`):

```js title="migrations/2025-01-10-create-users-and-tasks.js"
import { sql } from 'kysely';

export async function up(db) {
  await db.schema.createTable('users')
    .addColumn('id', 'serial', c => c.primaryKey())
    .addColumn('email', 'text', c => c.notNull().unique())
    .addColumn('created_at', 'timestamptz', c => c.notNull().defaultTo(sql`now()`))
    .execute();
  await db.schema.createTable('tasks')
    .addColumn('id', 'serial', c => c.primaryKey())
    .addColumn('user_id', 'integer', c => c.notNull().references('users.id').onDelete('cascade'))
    .addColumn('title', 'text', c => c.notNull())
    .addColumn('done', 'boolean', c => c.notNull().defaultTo(false))
    .execute();
  await db.schema.createIndex('tasks_user_id_idx').on('tasks').column('user_id').execute();
}

export async function down(db) {
  await db.schema.dropTable('tasks').execute();
  await db.schema.dropTable('users').execute();
}
```

```js title="scripts/migrate.js"
import { promises as fs } from 'node:fs';
import path from 'node:path';
import { Migrator, FileMigrationProvider } from 'kysely/migration';
import { createDb } from '../src/db.js';

const db = createDb({ connectionString: process.env.DATABASE_URL });
const migrator = new Migrator({
  db,
  provider: new FileMigrationProvider({
    fs, path, migrationFolder: path.join(import.meta.dirname, '../migrations'),
  }),
});
const { error, results } = await migrator.migrateToLatest();
for (const r of results ?? []) console.log(`${r.status}: ${r.migrationName}`);
await db.destroy();
if (error) { console.error(error); process.exit(1); }
```

Running it the first time printed `Success: 2025-01-10-create-users-and-tasks`; a second
run prints nothing, because the migration is already recorded. (In the Kysely release
used here, the migration classes are imported from the `kysely/migration` subpath; older
releases exported them from `kysely` directly. If an import fails, check your version's
docs.)

Constraints (`not null`, `unique`, foreign keys) belong in the database, not only in
your validation code: they hold even when two requests race or a script bypasses the API.

## Worked example: queries

```js title="queries.js"
const user = await db.insertInto('users').values({ email: 'ada@example.com' })
  .returning(['id', 'email']).executeTakeFirstOrThrow();
console.log(user);

await db.insertInto('tasks').values([
  { user_id: user.id, title: 'write migration' },
  { user_id: user.id, title: 'add index', done: true },
]).execute();

const q = db.selectFrom('tasks')
  .innerJoin('users', 'users.id', 'tasks.user_id')
  .select(['tasks.id', 'tasks.title', 'users.email'])
  .where('tasks.done', '=', false)
  .orderBy('tasks.id');
console.log(q.compile().sql);
console.log(await q.execute());

const res = await db.updateTable('tasks').set({ done: true }).where('id', '=', 1).executeTakeFirst();
console.log('updated rows:', res.numUpdatedRows);

try {
  await db.insertInto('users').values({ email: 'ada@example.com' }).execute();
} catch (e) { console.log('duplicate:', e.code, e.message); }
```

```text
{ id: 1, email: 'ada@example.com' }
select "tasks"."id", "tasks"."title", "users"."email" from "tasks" inner join "users" on "users"."id" = "tasks"."user_id" where "tasks"."done" = $1 order by "tasks"."id"
[ { id: 1, title: 'write migration', email: 'ada@example.com' } ]
updated rows: 1n
duplicate: 23505 duplicate key value violates unique constraint "users_email_key"
```

Look at the compiled SQL: the value `false` became **`$1`**. Kysely never splices values
into SQL text; it sends them separately as parameters. That is what makes it immune to
SQL injection for values. (Identifiers — table and column names — are a different story:
never build them from user input.)

`numUpdatedRows` is a `BigInt` (`1n`), since row counts can exceed 2^53.

## Transactions

When several writes must succeed or fail together, use a transaction. Everything inside
uses `trx` instead of `db`; if the callback throws, Kysely issues `ROLLBACK`:

```js
await db.transaction().execute(async (trx) => {
  const u = await trx.insertInto('users').values({ email: 'grace@example.com' })
    .returning('id').executeTakeFirstOrThrow();
  await trx.insertInto('tasks').values({ user_id: u.id, title: 'first task' }).execute();
});
```

Aggregates work as you would expect:

```js
const counts = await db.selectFrom('tasks')
  .select(({ fn }) => ['user_id', fn.countAll().as('n')])
  .groupBy('user_id').orderBy('user_id').execute();
```

One gotcha: with the real `pg` driver, `count(*)` returns a Postgres `bigint`, which `pg`
gives you as a **string** (`'2'`) to avoid precision loss. Convert with `Number()` when
you know it's small. (PGlite returned plain numbers in our run, one of the small ways an
emulated database can differ from the real one.)

## Escape hatch: raw SQL

For queries the builder doesn't express nicely, use the `sql` tag — still parameterized:

```js
import { sql } from 'kysely';
const { rows } = await sql`
  select date_trunc('day', created_at) as day, count(*)::int as n
  from tasks where user_id = ${userId}
  group by 1 order by 1`.execute(db);
```

## How It Actually Works

**The pool.** Opening a Postgres connection involves a TCP handshake, optional TLS, and
authentication — and each connection is a separate backend *process* on the server with
its own memory. So you don't connect per query. `pg.Pool` keeps up to `max` connections
open. A query borrows an idle connection, sends the query, and returns it to the pool.
If all are busy, the query waits in a queue. Transactions **pin** one connection for
their whole duration — which is why Kysely hands you a `trx` object: it's bound to that
connection, so `BEGIN`, your statements, and `COMMIT` all go over the same socket.

Pool sizing is a trade-off: total connections across all your Node processes must stay
under the server's `max_connections`. Ten processes with `max: 20` want 200 connections.
Beyond that point, poolers like PgBouncer multiplex many clients onto fewer server
connections.

**Parameters on the wire.** For parameterized queries `pg` uses Postgres's *extended
query protocol*: a `Parse` message with the SQL containing `$1, $2`, a `Bind` message with
the values sent separately, then `Execute`. The server never parses values as SQL, so a
value like `'; drop table users; --` is just a string.

**Non-blocking I/O.** Each connection is a TCP socket handled by the event loop, not the
thread pool. While a query runs on the server, your process serves other requests.
Rows arrive as bytes, and `pg` parses them into JavaScript values using type OIDs (int4 →
number, bool → boolean, timestamptz → `Date`, int8/numeric → string).

## Common mistakes

- **Creating a new pool per request** — connection exhaustion within minutes.
- **String-building SQL** with template literals (not the `sql` tag) → injection.
- **Forgetting `where`** on update/delete. Kysely will happily update every row.
- **Holding a transaction open across slow work** (HTTP calls) — it pins a connection
  and holds locks.
- **N+1 queries** — looping over 100 users and querying each one's tasks. Use a join or
  a single `where user_id in (...)`.
- **No indexes on foreign keys** you filter by (`tasks.user_id`).

## Exercise

1. Add a `projects` table and a `project_id` foreign key on `tasks` via a new migration.
   Write `down` too, and run up → down → up.
2. Implement `GET /projects/:id/stats` returning counts of done and open tasks in one
   query using `count(*) filter (where done)`.
3. Write a transfer-style operation: move all open tasks from one project to another and
   log the move in a `task_events` table, atomically. Force an error in the middle and
   confirm nothing changed.
4. Use `EXPLAIN` (via the `sql` tag) on the task-list query with and without the
   `user_id` index on a table of 100,000 rows. Compare the plans.
