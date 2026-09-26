---
description: "Level 2 of Node.js: Express 5, routing and middleware, REST design, Zod validation, error handling, PostgreSQL with Kysely, auth, testing, and logging."
---

# Level 2 · Intermediate <span class="level-badge">Building APIs</span>

Goal: build a backend service the way working teams do. You'll move from the raw
`node:http` server of Level 1 to Express 5, and then add, one lesson at a time, the parts
every production API needs: consistent resource design, validated input, a single error
path, a real database, password and token authentication, an automated test suite, and
structured logs.

Each lesson contributes a file to the level project, a multi-user tasks API. By module
10 you will assemble them and run a nine-test integration suite against a real Postgres
engine.

## Modules

1. [Express Fundamentals](01-express-fundamentals.md) — what Express adds to `node:http`, `req`/`res` helpers, and the app factory
2. [Routing & Middleware](02-routing-middleware.md) — route syntax in Express 5, routers per resource, and how the middleware stack runs
3. [REST API Design](03-rest-api-design.md) — URLs, methods, status codes, error bodies, cursor pagination, versioning, idempotency keys
4. [Validating Input with Zod](04-validation.md) — schemas for body, params, and query, and a reusable validation middleware
5. [Error Handling](05-error-handling.md) — operational vs programmer errors, one error middleware, and crash-safe processes
6. [PostgreSQL with Kysely](06-databases-postgres.md) — pools, parameterized queries, transactions, migrations, and alternatives
7. [Authentication: Sessions, JWT & Password Hashing](07-authentication.md) — scrypt, cookie sessions, bearer JWTs, and choosing between them
8. [Testing with node:test & Supertest](08-testing.md) — unit tests with mocks and HTTP integration tests against a database
9. [Structured Logging with Pino](09-logging.md) — JSON logs, levels, request ids, and redacting secrets
10. [Project — A REST API with Auth](10-project-rest-api-auth.md) — the complete tasks API with tests

## What you need before starting

- Level 1, especially the raw HTTP server (lesson 08) and configuration (lesson 09).
- Basic SQL: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `JOIN`, and what a primary key and
  foreign key are. The [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/)
  covers these.
- For running the API outside tests: a PostgreSQL server (local install or a container).
  The test suite itself runs without one, using an in-process Postgres build.

Package versions used while writing this level: Express 5, Zod 4, Kysely 0.29, pg 8,
Pino 10, Supertest 7. Minor versions move quickly; if an API looks different in your
installed version, check that package's changelog.
