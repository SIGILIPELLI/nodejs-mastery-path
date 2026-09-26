---
description: "ES modules vs CommonJS in Node: how Node decides a file's module type, import/export syntax, interop in both directions, and module resolution."
---

# 03 · Modules: ES Modules & CommonJS

Node has two module systems living side by side:

- **ES modules (ESM)** — the JavaScript standard: `import` / `export`. This is what you
  should write for new code, and what this course uses throughout.
- **CommonJS (CJS)** — Node's original system: `require()` / `module.exports`. A huge
  amount of the npm ecosystem still ships CommonJS, so you need to read it and interoperate
  with it.

## How Node decides which system a file uses

Node looks at the file extension first, then at the nearest `package.json`:

| File | Treated as |
|------|-----------|
| `*.mjs` | always ESM |
| `*.cjs` | always CommonJS |
| `*.js` with nearest `package.json` containing `"type": "module"` | ESM |
| `*.js` with `"type": "commonjs"` or no `type` field | CommonJS (with a fallback, below) |

Recent Node versions also *detect* ESM syntax: if a `.js` file with no `type` field
contains `import`/`export` and fails to parse as CommonJS, Node retries it as ESM. Do not
rely on that — it costs a double parse and is ambiguous to readers and tools. Set
`"type": "module"` explicitly:

```bash
npm pkg set type=module
```

## ESM syntax you will actually use

```js title="math.mjs"
export const PI = 3.14159;
export function area(r) { return PI * r * r; }
export default function describe() { return 'geometry helpers'; }
```

```js title="main.mjs"
import describe, { area } from './math.mjs';     // default + named
import * as math from './math.mjs';              // namespace object
import { readFile } from 'node:fs/promises';     // built-in, with node: prefix

const config = JSON.parse(await readFile(new URL('./config.json', import.meta.url), 'utf8'));
const { default: lazy } = await import('./math.mjs'); // dynamic import, returns a promise
```

Rules that trip people up in ESM:

- **Relative imports need the file extension**: `'./math.js'`, not `'./math'`. ESM
  resolution in Node does not guess extensions or look for `index.js`.
- **No `__dirname`, `__filename`, or `require`** by default. Use `import.meta.dirname`
  and `import.meta.filename` (available in current Node releases), or `import.meta.url`
  with `new URL(...)`.
- **Top-level `await` works** — handy for loading config at startup.
- ESM is always in **strict mode**.

## CommonJS syntax you need to read

```js title="legacy.cjs"
const path = require('node:path');
exports.greet = (name) => `hello, ${name}`;
exports.version = 1;
// or: module.exports = { greet, version };
```

`require` is synchronous and can be called anywhere, including inside `if` blocks.
`__dirname` and `__filename` exist in every CommonJS module.

## Interop in both directions

**ESM importing CommonJS** works out of the box. `module.exports` becomes the default
export, and Node statically analyzes the CJS source to offer named exports where it can:

```js title="main.mjs"
import legacy, { greet } from './legacy.cjs';
import { createRequire } from 'node:module';

const require = createRequire(import.meta.url);   // a real require() inside ESM
const again = require('./legacy.cjs');

console.log(greet('esm'), legacy.version, again === legacy);
```

```text
hello, esm 1 true
```

`again === legacy` is `true`: both routes hit the same cached module instance.

**CommonJS requiring ESM** used to be impossible (you had to use `await import()`).
Current Node releases support `require()` of an ES module as long as that module graph
does not use top-level `await`:

```js title="old.cjs"
const { area } = require('./math.mjs');
console.log('require(esm):', area(1));
```

```text
require(esm): 3.14159
```

On older Node versions this throws `ERR_REQUIRE_ESM`. If you publish a library that must
support older runtimes, keep that in mind.

## Worked example: live bindings

ESM exports are **live bindings**, not copies:

```js title="counter.mjs"
export let count = 0;
export function inc() { count++; }
```

```js title="use-counter.mjs"
import { count, inc } from './counter.mjs';
inc(); inc();
console.log(count); // 2
```

With CommonJS, `const { count } = require('./counter')` would copy the number `0` at
require time and never see later changes. In ESM, `count` is a read-only view onto the
exporting module's variable. You cannot assign to it from the importer (`count = 5`
throws a `TypeError`).

## How It Actually Works

**Resolution.** For a bare specifier like `import express from 'express'`, Node walks up
the directory tree from the importing file: `./node_modules/express`,
`../node_modules/express`, `../../node_modules/express`, and so on to the filesystem
root. In the package it finds, it reads `package.json`:

- If there is an `"exports"` field, it is authoritative. It maps subpaths and
  *conditions* (`"import"`, `"require"`, `"node"`, `"default"`) to files. Anything not
  listed is not importable — `import 'pkg/internal/x.js'` fails with
  `ERR_PACKAGE_PATH_NOT_EXPORTED`.
- Otherwise Node falls back to `"main"` (and for CommonJS, to `index.js`).

This is how a single package can ship both formats ("dual packages"): `"exports":
{ "import": "./esm/index.js", "require": "./cjs/index.cjs" }`.

**Loading.** CommonJS wraps each file in a function
`(function (exports, require, module, __filename, __dirname) { ... })` and runs it
synchronously; that's where those "globals" come from. The result is stored in
`require.cache` keyed by absolute path, so the second `require` returns the same object.

ESM loads in three phases: **construction** (fetch and parse every module in the graph,
finding `import` statements statically), **instantiation** (wire each import to the
exporting module's binding — the live-binding mechanism), and **evaluation** (run module
bodies, dependencies first). Because imports are known before any code runs, tools can
tree-shake them and Node can report a misspelled named import before executing anything.
ESM instances are cached in a module map keyed by URL (`file:///...`), so every importer
shares one instance.

## Common mistakes

- **Missing extensions in ESM imports** → `ERR_MODULE_NOT_FOUND`. Write `./utils.js`.
- **Using `__dirname` in ESM** → `ReferenceError`. Use `import.meta.dirname`.
- **Mixing `require` and `import` in the same `.js` file.** A file is one or the other.
- **Default-import confusion with CJS packages.** Some CJS packages set
  `exports.default`; importing them from ESM can give you `{ default: fn }`. Check what
  you received with `console.log`.
- **Relying on module-level state as "per request" state.** Modules are singletons per
  process; a module-level variable is shared by every request.

## Exercise

1. Create a project with `"type": "module"`. Write `lib/strings.js` exporting `slugify`
   (default) and `capitalize` (named), and import both from `index.js`.
2. Add a `legacy.cjs` that exports an object with `module.exports = {...}`. Import it from
   ESM, then `require()` your ESM file from another `.cjs` file. Note which named exports
   are visible.
3. Add an `"exports"` field to your `package.json` exposing only `.` and `./strings`.
   Install your package into another folder with `npm install ../path` and confirm that
   deep imports outside `exports` are rejected.
