---
description: "Design REST APIs that age well: resource URLs, method semantics, status codes, error bodies, cursor pagination, filtering, versioning, and idempotency."
---

# 03 · REST API Design

Express will happily let you build `POST /getUserOrdersNow`. Nothing stops you except
the clients who have to use it. Good API design is mostly about **predictability**: a
developer who has seen two of your endpoints should be able to guess the third. This
lesson collects the conventions that make an HTTP JSON API predictable, and shows how to
implement the trickier ones (pagination, idempotency) in Node.

## Resources and URLs

Model **nouns** (resources), and let HTTP methods be the verbs.

| Operation | Method + path | Success status |
|-----------|---------------|----------------|
| List tasks | `GET /tasks` | 200 |
| Create a task | `POST /tasks` | 201 + `Location: /tasks/17` |
| Read one | `GET /tasks/17` | 200 |
| Partial update | `PATCH /tasks/17` | 200 (with body) or 204 |
| Replace | `PUT /tasks/17` | 200 or 204 |
| Delete | `DELETE /tasks/17` | 204 |
| Sub-collection | `GET /projects/3/tasks` | 200 |

Guidelines:

- Plural nouns, lowercase, hyphens for multi-word names: `/billing-accounts`.
- Nest only one level deep for ownership (`/projects/3/tasks`); beyond that, use filters
  (`/tasks?projectId=3&assignee=9`).
- For real actions that aren't CRUD, a sub-resource verb is acceptable and clearer than
  contortions: `POST /orders/42/cancel`.
- Use the same field naming everywhere (`camelCase` is most natural for JS clients).

## Method semantics that matter

- **Safe** methods (`GET`, `HEAD`) must not change state. Caches, crawlers, and link
  prefetchers will call them freely.
- **Idempotent** methods (`GET`, `PUT`, `DELETE`) produce the same server state whether
  called once or five times. Clients and proxies may retry them after a network error.
- `POST` is neither, which is why retrying a "create payment" `POST` is dangerous (see
  idempotency keys below).
- `PATCH` sends only the fields to change. `PUT` sends the whole representation; missing
  fields mean "remove/reset".

## Status codes: a practical subset

| Code | Use for |
|------|---------|
| 200 OK | Successful read/update with a body |
| 201 Created | Resource created; include `Location` |
| 202 Accepted | Work queued, not done yet (Level 4 queues) |
| 204 No Content | Success, no body (DELETE) |
| 400 Bad Request | Malformed JSON or failed validation |
| 401 Unauthorized | Not authenticated (missing/invalid credentials) |
| 403 Forbidden | Authenticated but not allowed |
| 404 Not Found | No such resource — *or* one the caller may not know exists |
| 409 Conflict | Duplicate unique value, version conflict |
| 413 / 415 | Body too large / unsupported content type |
| 422 Unprocessable Content | Some teams use this for validation errors instead of 400; pick one |
| 429 Too Many Requests | Rate limited; include `Retry-After` |
| 500 / 503 | Server bug / temporarily unavailable |

Returning **404 instead of 403** for another user's resource avoids confirming that it
exists. The Level 2 project does this for tasks.

## One error format

Pick a single error shape and use it for every failure, so clients can write one handler:

```json
{
  "error": "validation failed",
  "details": [
    { "path": "body.title", "message": "Too small: expected string to have >=1 characters" }
  ]
}
```

A standardized alternative is **Problem Details** (RFC 9457), served as
`application/problem+json` with `type`, `title`, `status`, `detail`, and extension
fields. Either is fine; inconsistency is not.

## Pagination

Never return an unbounded list. Two main styles:

**Offset pagination** — `GET /tasks?limit=20&offset=40`. Simple, allows jumping to
page N, but `OFFSET 100000` makes the database scan and discard 100,000 rows, and
inserts between requests shift items so clients see duplicates or gaps.

**Cursor (keyset) pagination** — the client passes the last id it saw:
`GET /tasks?limit=20&cursor=4812`. The query becomes `WHERE id > 4812 ORDER BY id LIMIT
21`, which an index answers directly regardless of depth, and new inserts don't shift
pages. You lose "jump to page 37". This is what the Level 2 project implements:

```js
router.get('/', validate({ query: listQuery }), async (req, res) => {
  const { done, limit, cursor } = req.valid.query;
  let q = db.selectFrom('tasks').select(columns)
    .where('user_id', '=', req.user.id)
    .orderBy('id').limit(limit + 1);          // fetch one extra to learn if there's more
  if (done !== undefined) q = q.where('done', '=', done);
  if (cursor) q = q.where('id', '>', cursor);
  const rows = await q.execute();
  const hasMore = rows.length > limit;
  const data = rows.slice(0, limit);
  res.json({ data, nextCursor: hasMore ? data.at(-1).id : null });
});
```

Wrapping lists in `{ data, nextCursor }` rather than returning a bare array leaves room
to add metadata later without breaking clients. For sorts on non-unique columns
(e.g. `created_at`), the cursor must include a tiebreaker: `(created_at, id)`.

## Filtering, sorting, sparse fields

- Filters as query params: `?done=false&assignee=9`.
- Sorting: `?sort=-createdAt,title` (leading `-` for descending). **Allowlist** sortable
  fields; never interpolate a client-supplied column name into SQL.
- Cap `limit` (e.g. max 100) on the server.

## Versioning

Additive changes (new optional fields, new endpoints) don't need a new version. Breaking
changes (removing/renaming fields, changing types or meaning) do. The simplest scheme is a
URL prefix: `/v1/tasks`. Header-based versioning is possible but harder to test with a
browser or `curl`. Whatever you choose, **clients must ignore unknown fields** so you can
add fields freely.

## Idempotency keys for unsafe retries

For `POST` endpoints where a duplicate would hurt (payments, orders, emails), accept an
`Idempotency-Key` header. Store the key with the result; on a repeat, return the stored
result instead of acting again:

```js
router.post('/payments', async (req, res) => {
  const key = req.get('idempotency-key');
  if (!key) return res.status(400).json({ error: 'Idempotency-Key header required' });

  const previous = await db.selectFrom('idempotency').selectAll()
    .where('key', '=', key).where('user_id', '=', req.user.id).executeTakeFirst();
  if (previous) return res.status(previous.status).json(previous.body);

  const payment = await createPayment(req.valid.body);   // the real side effect
  await db.insertInto('idempotency')
    .values({ key, user_id: req.user.id, status: 201, body: JSON.stringify(payment) })
    .execute();
  res.status(201).json(payment);
});
```

This sketch has a race when two identical requests arrive together; production versions
insert the key first inside a transaction (with a unique constraint) and mark it
completed afterwards.

## How It Actually Works

REST conventions work because **intermediaries understand HTTP semantics**. A CDN or
browser cache stores responses to `GET` according to `Cache-Control` and validates them
with `ETag`/`If-None-Match`; it would never cache a `POST`. HTTP client libraries and
proxies may automatically retry idempotent requests on a dropped connection but not
`POST`. Load balancers and monitoring classify errors by status code class — a 500
counts against your availability SLO, a 400 doesn't. When you use methods and codes
correctly, all this infrastructure does the right thing for free. When you tunnel
everything through `POST` with `200 { "success": false }`, each layer is blind.

Keyset pagination is fast because a B-tree index on `id` is sorted: the database seeks
directly to the first key greater than the cursor and reads the next `limit` entries.
Offset pagination has to walk the index from the beginning and count.

## Common mistakes

- **Verbs in URLs** for plain CRUD (`/createTask`).
- **200 for everything**, with errors in the body.
- **Unbounded lists** and client-controlled page sizes without a cap.
- **Leaking internal IDs or DB errors** in responses.
- **Breaking changes without a version**, e.g. renaming a field in place.
- **Inconsistent shapes** — some endpoints return arrays, others `{ items }`, others
  `{ data }`.

## Exercise

1. Design (on paper) the endpoints for a library system with books, members, and loans,
   including "return a book" and "list overdue loans". Give methods, paths, success codes,
   and two error cases each.
2. Implement `sort` for `GET /tasks` supporting `createdAt` and `title` with `-` for
   descending, using an allowlist.
3. Change the project's cursor to an opaque string: base64url-encode `{ id }` so clients
   don't depend on its structure. Decode and validate it on the way in.
4. Make the idempotency sketch race-free using a unique constraint on `(user_id, key)`
   and a status column (`in_progress` / `completed`).
