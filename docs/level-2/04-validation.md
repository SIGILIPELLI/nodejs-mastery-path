---
description: "Validate API input with Zod: schemas for body, params and query, coercion and defaults, readable error details, and a reusable Express middleware."
---

# 04 · Validating Input with Zod

Every value that arrives over HTTP is untrusted: the body might be missing fields, the
`id` param might be `"abc"`, the query might contain an array where you expected a
string, and an attacker might add `"isAdmin": true` to see what happens. Validation turns
"whatever the client sent" into a value with a known shape — or a clear 400 error —
**at the edge**, so the rest of your code can trust its inputs.

Writing these checks by hand (as in Level 1) gets unwieldy fast. This course uses
**Zod**, a schema library where one declaration gives you runtime validation, parsing
(coercion, trimming, defaults), and — in TypeScript — a static type. Alternatives you
will meet include Valibot, ArkType, Joi, Yup, and JSON-Schema validators such as Ajv
(which Fastify uses natively). The ideas are identical.

```bash
npm install zod
```

The examples use Zod 4's API (`z.email()` as a top-level format).

## Schemas describe and transform

```js title="schemas.js"
import { z } from 'zod';

const Address = z.object({
  city: z.string().min(1),
  postcode: z.string().regex(/^\d{5}$/, 'postcode must be 5 digits'),
});

const CreateUser = z.object({
  email: z.email(),
  name: z.string().trim().min(1).max(100),
  age: z.coerce.number().int().min(13).optional(),
  role: z.enum(['member', 'admin']).default('member'),
  tags: z.array(z.string()).max(10).default([]),
  address: Address.optional(),
});

const good = CreateUser.parse({ email: 'ada@example.com', name: '  Ada ', age: '36' });
console.log(good);
```

```text
{
  email: 'ada@example.com',
  name: 'Ada',
  age: 36,
  role: 'member',
  tags: []
}
```

`parse` returned a **new** object: the name was trimmed, `'36'` was coerced to `36`, and
defaults were filled in. Use the parsed output, not the original input.

## Handling failure

`parse` throws a `ZodError`; `safeParse` returns `{ success, data }` or
`{ success: false, error }`, which is easier to turn into an HTTP response:

```js
const bad = CreateUser.safeParse({
  email: 'not-an-email', name: '', age: 9, role: 'owner',
  address: { city: 'Pune', postcode: '4110' }, isAdmin: true,
});
console.log(bad.success);
for (const issue of bad.error.issues) console.log(issue.path.join('.'), '-', issue.message);
```

```text
false
email - Invalid email address
name - Too small: expected string to have >=1 characters
age - Too small: expected number to be >=13
role - Invalid option: expected one of "member"|"admin"
address.postcode - postcode must be 5 digits
```

All problems are reported at once, each with a `path`, so a form can highlight every bad
field. And `isAdmin`? `z.object` **strips unknown keys** by default, so it silently
disappears from the output. That is a security feature: extra properties never reach your
database layer. If you'd rather reject them, use `z.strictObject({...})` (or `.strict()`),
which reports `Unrecognized key: "isAdmin"`.

## Worked example: a validation middleware

Rather than calling `safeParse` in every handler, write one middleware that validates any
part of the request against a schema and stores the parsed result. This is the version
used in the Level 2 project:

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

Using it:

```js
const idParams = z.object({ id: z.coerce.number().int().positive() });
const listQuery = z.object({
  done: z.enum(['true', 'false']).transform(v => v === 'true').optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.coerce.number().int().positive().optional(),
});
const updateBody = z.object({
  title: z.string().trim().min(1).max(200).optional(),
  done: z.boolean().optional(),
}).refine(b => Object.keys(b).length > 0, 'provide at least one field');

router.get('/', validate({ query: listQuery }), listTasks);
router.patch('/:id', validate({ params: idParams, body: updateBody }), updateTask);
```

A request `PATCH /tasks/abc` produces a 400 with a `params.id` detail before the handler
runs. This middleware stops at the first failing part (params before body, in the order
you list them); collecting issues from every part first is a small change you can make in
the exercises if your clients want all errors at once.

Why `req.valid` instead of overwriting `req.query`? In Express 5, `req.query` is a
getter computed from the URL, so assigning to it doesn't stick. Keeping parsed data in a
separate property also makes it obvious in handlers which values have been validated.

Notice the query schema: query-string values are always strings (or arrays of strings),
so booleans need an explicit `'true'`/`'false'` mapping. `z.coerce.boolean()` would turn
the string `'false'` into `true`, because any non-empty string is truthy.

## Validating configuration with the same tool

The Level 1 config module can shrink to a schema. From the project:

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

A missing variable fails at startup with a readable list:

```text
Invalid configuration:
  PORT: Invalid input: expected number, received NaN
  DATABASE_URL: Invalid input: expected string, received undefined
  JWT_SECRET: JWT_SECRET must be at least 32 characters
```

## How It Actually Works

A Zod schema is a tree of objects, each knowing how to check and transform one kind of
value. `z.object({...})` holds a map of child schemas; calling `safeParse(input)` walks
the tree recursively:

1. Check the input's runtime type (`typeof`, `Array.isArray`, etc.).
2. For objects, run each child schema on `input[key]`, building a *new* output object —
   keys not in the shape are never copied (that's the stripping).
3. Run checks (`min`, `regex`, `int`...) and record an **issue** with the current path
   for each failure, instead of stopping at the first.
4. Apply transforms (`trim`, `coerce`, `default`, `.transform(fn)`) to produce the output.
5. Run `refine` callbacks on the assembled value.

If any issues were collected, the result is a failure listing them all. `coerce` simply
wraps the input with a JavaScript conversion (`Number(input)`, `String(input)`) before
checking, which explains the `z.coerce.boolean()` surprise.

Validation at the boundary is the implementation of an old rule: **parse, don't
validate**. Instead of checking a value and continuing to pass the loose original around,
you convert it once into a well-typed value, and every function downstream receives only
that.

## Common mistakes

- **Validating but then using `req.body`** instead of the parsed output — you lose
  trimming, defaults, and stripping.
- **Trusting TypeScript types for request data.** Types vanish at runtime; only runtime
  validation protects you.
- **`z.coerce.boolean()` on query strings.**
- **Unbounded strings and arrays** — always `max()` them to protect memory and the DB.
- **Leaking validation of secrets** — don't echo submitted passwords back in error
  details.
- **Validating only the body** while `params` and `query` flow straight into queries.

## Exercise

1. Write a schema for `POST /events` with `title`, `startsAt` (ISO date string converted
   to a `Date`), `endsAt` (must be after `startsAt` — use `refine` on the object), and an
   optional `attendees` array of emails (max 50, deduplicated via `transform`).
2. Add `validate({ query })` to `GET /tasks` supporting `sort` as an enum of allowed
   values.
3. Switch one schema to `z.strictObject` and write a test proving that unknown keys
   produce a 400.
4. Measure: `safeParse` a valid object 100,000 times in a loop with `performance.now()`
   around it. Is validation cost meaningful compared to a database round trip?
