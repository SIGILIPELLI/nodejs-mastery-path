---
description: "TypeScript for Node services: running .ts files via type stripping, erasable-only syntax, tsc as a type checker, and typed Express with Zod."
---

# 03 · TypeScript for Node

Most teams building Node services at scale use **TypeScript**: JavaScript plus a static
type system that catches mistakes — wrong argument types, misspelled properties, missing
`null` checks — before the code runs, and makes refactoring large codebases far less
scary. This lesson focuses on the Node-specific side: how TypeScript code actually gets
executed, how to configure it for ES modules, and how to type an Express service without
fighting the framework. For the language itself, see the
[TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/).

## Two jobs: type checking and running

TypeScript tooling does two separate things:

1. **Type checking** — `tsc` analyzes your code and reports errors. It is the only step
   that knows about types.
2. **Producing runnable JavaScript** — removing type annotations (and, for some syntax,
   transforming code).

Historically you compiled with `tsc` to a `dist/` folder, or ran code through `ts-node`,
`tsx`, or a bundler. Current Node releases can do step 2 themselves: **`node file.ts`
works** by stripping type annotations on the fly. (Older releases needed a flag such as
`--experimental-strip-types`; check your version.) Type checking remains `tsc`'s job.

```bash
npm install -D typescript @types/node
```

## Running .ts files directly

```ts title="src/money.ts"
export type Currency = 'USD' | 'EUR' | 'INR';

export interface Money {
  readonly amountMinor: number;   // integer minor units (cents, paise)
  readonly currency: Currency;
}

export function add(a: Money, b: Money): Money {
  if (a.currency !== b.currency) {
    throw new Error(`cannot add ${a.currency} to ${b.currency}`);
  }
  return { amountMinor: a.amountMinor + b.amountMinor, currency: a.currency };
}

export function format(m: Money): string {
  return new Intl.NumberFormat('en', { style: 'currency', currency: m.currency })
    .format(m.amountMinor / 100);
}
```

```ts title="src/main.ts"
import { add, format, type Money } from './money.ts';

const price: Money = { amountMinor: 1999, currency: 'USD' };
const shipping: Money = { amountMinor: 500, currency: 'USD' };
console.log(format(add(price, shipping)));
```

```text
$ node src/main.ts
$24.99
```

Note the import: `'./money.ts'` **with the `.ts` extension**, because Node resolves the
file exactly as written (ESM rules from Level 1, lesson 03). And `type Money` is imported
with the `type` modifier, marking it as erased at runtime.

### Only "erasable" syntax

Node's type stripping replaces type annotations with whitespace; it doesn't transform code.
TypeScript features that **generate JavaScript** therefore don't work:

```ts title="src/enum.ts"
enum Status { Active, Disabled }
```

```text
$ node src/enum.ts
SyntaxError [ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX]: TypeScript enum is not supported in strip-only mode
```

The same applies to `namespace` blocks with values, constructor parameter properties
(`constructor(private db: Db)`), and legacy decorators with metadata. Use union types
(`'active' | 'disabled'`) or `as const` objects instead of enums, and explicit class
fields instead of parameter properties. The compiler option `erasableSyntaxOnly` makes
`tsc` flag these, so editors warn you before Node does.

## Type checking with tsc

```json title="tsconfig.json"
{
  "compilerOptions": {
    "target": "es2023",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "noEmit": true,
    "allowImportingTsExtensions": true,
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

- `module`/`moduleResolution: "nodenext"` — follow Node's real ESM/CJS rules, including
  `"type": "module"` and `exports` maps.
- `noEmit` — `tsc` only checks; Node runs the `.ts` files.
- `allowImportingTsExtensions` — permit `./money.ts` imports.
- `verbatimModuleSyntax` — requires `import type` for type-only imports, so what you write
  is exactly what gets erased.
- `strict` — non-negotiable for new code.

**Type stripping does not check types.** A file with type errors still runs:

```ts title="src/bad.ts"
import { add, type Money } from './money.ts';
const a: Money = { amountMinor: 100, currency: 'USD' };
add(a, { amountMinor: '5', currency: 'GBP' });
```

```text
$ npx tsc
src/bad.ts(3,10): error TS2322: Type 'string' is not assignable to type 'number'.
src/bad.ts(3,28): error TS2322: Type '"GBP"' is not assignable to type 'Currency'.
$ node src/bad.ts
Error: cannot add USD to GBP
```

`tsc` caught both mistakes; Node happily ran the file until the runtime check threw.
So `tsc --noEmit` must run in CI (and ideally in a pre-commit hook or your editor).
Add scripts:

```json
{ "scripts": { "typecheck": "tsc", "start": "node src/server.ts", "test": "node --test" } }
```

If you publish a library or want the smallest possible startup, you can still compile to
JavaScript with `tsc` (with `rewriteRelativeImportExtensions` turning `.ts` imports into
`.js`) and run the output.

## Worked example: typed Express with Zod

Types and runtime validation should come from **one source**. With Zod, the schema is the
source and the type is derived:

```ts title="src/app.ts"
import express, { type Request, type Response, type NextFunction } from 'express';
import { z } from 'zod';

const CreateProduct = z.object({
  name: z.string().trim().min(1).max(120),
  priceMinor: z.number().int().nonnegative(),
  tags: z.array(z.string()).default([]),
});
type CreateProduct = z.infer<typeof CreateProduct>;   // the static type, derived from the schema

interface Product extends CreateProduct { id: number }

const products = new Map<number, Product>();
let nextId = 1;

// Typed middleware that validates the body and passes parsed data via res.locals
function validateBody<S extends z.ZodType>(schema: S) {
  return (req: Request, res: Response<unknown, { body: z.infer<S> }>, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      res.status(400).json({ error: 'validation failed', issues: result.error.issues });
      return;
    }
    res.locals.body = result.data;
    next();
  };
}

export const app = express();
app.use(express.json());

app.post('/products', validateBody(CreateProduct), (req, res: Response<Product, { body: CreateProduct }>) => {
  const product: Product = { id: nextId++, ...res.locals.body };
  products.set(product.id, product);
  res.status(201).json(product);
});

app.get('/products/:id', (req: Request<{ id: string }>, res: Response<Product | { error: string }>) => {
  const product = products.get(Number(req.params.id));
  if (!product) {
    res.status(404).json({ error: 'not found' });
    return;
  }
  res.json(product);
});
```

```text
$ npx tsc && node src/server.ts
{ id: 1, name: 'Desk lamp', priceMinor: 2599, tags: [] }
{ id: 1, name: 'Desk lamp', priceMinor: 2599, tags: [] }
```

(`tsc` exited with no errors; the run used Express 5 with `@types/express`.) Points to
notice:

- `z.infer<typeof CreateProduct>` gives the exact type of the *parsed* value, including
  the default for `tags`. Change the schema and every use of the type updates.
- `Request<{ id: string }>` types `req.params`; route params are strings, and the type
  says so.
- `Response<Body, Locals>` types what `res.json` accepts and what `res.locals` holds, so
  the validated body flows to the handler with a real type.
- Request data is `unknown` until validated. Resist `req.body as CreateProduct` — a type
  assertion is a promise to the compiler that nothing checks at runtime.

## How It Actually Works

Node's TypeScript support is built on **Amaro**, a small wrapper around the SWC parser
compiled to WebAssembly. When Node loads a `.ts`/`.mts`/`.cts` file, it parses it and
replaces every type-only construct (annotations, interfaces, `type` aliases, `import type`)
with **whitespace of the same length**. Because positions don't move, line and column
numbers in stack traces match your source without source maps — note how the `bad.ts`
error pointed at `money.ts:10`. Files inside `node_modules` are deliberately not
stripped: packages are expected to ship JavaScript.

`tsc` works entirely separately: it builds a program from your `include` files, resolves
imports with the configured module resolution, loads declaration files (`@types/node`,
`@types/express`, or types bundled with packages like Zod), and runs the type checker. It
never affects what Node executes when `noEmit` is set.

Types are **erased**: at runtime there is no `Money` interface to check against. That is
exactly why HTTP input still needs Zod (or similar) even in a fully typed codebase.

## Common mistakes

- **Assuming `node file.ts` type-checks.** It doesn't. Run `tsc` in CI.
- **Enums, parameter properties, and namespaces** in code meant to run via type stripping.
- **Importing `./x.js` in source that has `./x.ts`** under strip mode, or omitting
  extensions entirely.
- **`as` casts on request data** instead of validation.
- **`any` creep** — enable `strict` and `noImplicitAny` (included in `strict`) from day one;
  prefer `unknown` for values you haven't checked.
- **Mismatched `@types/*` versions** — e.g. `@types/node` for a different major than you
  run. Match them to your runtime.

## Exercise

1. Convert the Level 2 `validate.js` and `errors.js` to TypeScript and make `tsc` pass
   with `strict`. Type `HttpError` so `status` is a union of the codes you actually use.
2. Replace an `enum` in some existing code (or invent one) with an `as const` object and
   a derived union type: `type Role = (typeof Roles)[keyof typeof Roles]`.
3. Add `npm run typecheck` to a CI workflow and make the build fail on type errors.
4. Deliberately introduce a type error and confirm the app still starts with `node`, then
   explain to a teammate why CI must run `tsc`.
