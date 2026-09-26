---
description: "Level 1 of Node.js: the V8 + libuv runtime, installing with nvm, ES modules and CommonJS, npm, fs and path, async patterns, events, raw HTTP, and config."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: understand what the `node` binary actually does with your code, and get fluent with
the built-in modules every backend relies on — before any framework enters the picture.
By the end of this level you will have written a dependency-free JSON API on `node:http`
and shipped an installable command-line tool.

The level deliberately stays close to the metal. When Express shows up in Level 2 you
should recognize `req`, `res`, body parsing, and routing as conveniences over things you
already built by hand.

## Modules

1. [What Node.js Is: V8 + libuv](01-what-is-node.md) — the runtime's two engines, the single JavaScript thread, and the hidden thread pool
2. [Installing Node & Managing Versions](02-installing-node-nvm.md) — LTS vs Current, nvm/fnm, `.nvmrc`, and `engines`
3. [Modules: ES Modules & CommonJS](03-modules-esm-cjs.md) — how Node picks a module system, interop both ways, and module resolution
4. [npm & package.json](04-npm-package-json.md) — dependencies, semver, the lockfile, scripts, and how `node_modules` is laid out
5. [Files & Paths: fs and path](05-fs-and-path.md) — promise-based fs, error codes, atomic writes, and path safety
6. [Async Patterns: Callbacks to async/await](06-async-patterns.md) — a Node-focused refresher: promisify, parallelism, cancellation, unhandled rejections
7. [Events & EventEmitter](07-events-eventemitter.md) — synchronous dispatch, the `'error'` event, and async iteration over events
8. [A Raw HTTP Server with node:http](08-raw-http-server.md) — routing, body parsing, and status codes with zero dependencies
9. [Environment Variables & Configuration](09-env-config.md) — `process.env`, `--env-file`, and a fail-fast config module
10. [Project — A Real CLI Tool](10-project-cli-tool.md) — `logstat`, a streaming access-log analyzer with flags, exit codes, and tests

## What you need before starting

- Working knowledge of modern JavaScript: `const`/`let`, arrow functions, destructuring,
  classes, template literals, and the basics of promises. The
  [JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/)
  covers these if you need them.
- A terminal you are comfortable in, and `curl` for poking at HTTP servers.
- A current Node release (lesson 02 shows how to install one). All code uses ES modules.

Keep the `logstat` project from module 10 — its structure (pure logic in `src/`, I/O at
the edges, tests beside it) is the same shape you will use for the API in Level 2.
