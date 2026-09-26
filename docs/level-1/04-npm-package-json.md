---
description: "npm and package.json in depth: dependencies vs devDependencies, semver ranges, the lockfile, npm scripts, npx, and how npm builds node_modules."
---

# 04 · npm & package.json

`package.json` is the manifest of a Node project: its name, how it runs, what it
depends on, and which Node versions it supports. npm (bundled with Node) reads it to
install dependencies and run scripts. Alternatives such as pnpm and Yarn read the same
file; the concepts here transfer directly.

## A realistic package.json

```json title="package.json"
{
  "name": "orders-api",
  "version": "0.3.0",
  "private": true,
  "type": "module",
  "main": "src/server.js",
  "engines": { "node": ">=22" },
  "scripts": {
    "start": "node src/server.js",
    "dev": "node --watch --env-file=.env src/server.js",
    "test": "node --test",
    "lint": "eslint .",
    "check": "npm run lint && npm test"
  },
  "dependencies": {
    "express": "^5.1.0",
    "pino": "^10.0.0"
  },
  "devDependencies": {
    "eslint": "^9.0.0",
    "supertest": "^7.0.0"
  }
}
```

Field by field:

- `"private": true` prevents an accidental `npm publish` of an application.
- `"type": "module"` makes `.js` files ES modules (lesson 03).
- `scripts` are named shell commands. `npm start` and `npm test` are shortcuts; anything
  else runs with `npm run <name>`.
- `dependencies` are needed at runtime. `devDependencies` are only needed to develop,
  test, or build. In production installs (`npm ci --omit=dev`) they are skipped, which
  shrinks your container image.

## Semantic versioning and ranges

Versions are `MAJOR.MINOR.PATCH`. By convention, a major bump means breaking changes,
minor means new backward-compatible features, patch means fixes. Ranges in
`package.json` say which versions you accept:

| Range | Accepts | Notes |
|-------|---------|-------|
| `^5.1.0` | `>=5.1.0 <6.0.0` | npm's default when you `npm install` |
| `~5.1.0` | `>=5.1.0 <5.2.0` | patches only |
| `5.1.0` | exactly that | use `npm install --save-exact` |
| `^0.4.2` | `>=0.4.2 <0.5.0` | for `0.x`, caret is stricter: minor bumps may break |

Semver is a promise made by humans, and it's sometimes broken. That's the job of the
lockfile.

## The lockfile

`package-lock.json` records the **exact** version, download URL, and integrity hash of
every package in the tree — including dependencies of dependencies. Commit it.

- `npm install` resolves ranges, may update the lockfile, and installs.
- `npm ci` installs *exactly* what the lockfile says, deletes any existing
  `node_modules` first, and fails if `package.json` and the lockfile disagree. Use it in
  CI and Docker builds for reproducibility.

## Day-to-day commands

```bash
npm install express            # add a runtime dependency
npm install -D vitest          # add a dev dependency
npm uninstall express          # remove
npm outdated                   # what's behind, and by how much
npm update                     # move within your declared ranges
npm ls pino                    # why is this in my tree, and which version?
npm explain pino               # who depends on it
npm audit                      # known vulnerabilities (Level 3 lesson 09)
npx eslint .                   # run a binary from node_modules/.bin (or fetch it)
```

## Worked example: scripts with pre/post hooks and arguments

```json
{
  "scripts": {
    "pretest": "npm run lint",
    "test": "node --test",
    "lint": "eslint src",
    "db:migrate": "node scripts/migrate.js"
  }
}
```

- `npm test` automatically runs `pretest` first. (`pre`/`post` hooks exist for any
  script name.)
- Pass extra arguments after `--`: `npm test -- --test-name-pattern=orders` runs
  `node --test --test-name-pattern=orders`.
- Scripts run with `node_modules/.bin` on the `PATH`, so `eslint` works without `npx`.
- Inside a script, npm exposes package fields as environment variables, e.g.
  `npm_package_version`.

## How It Actually Works

**Building the tree.** npm reads your direct dependencies, fetches metadata from the
registry (`registry.npmjs.org` by default), picks the highest version satisfying each
range, and recurses into their dependencies. Then it lays the result out on disk. Since
npm 3 the layout is **hoisted and deduplicated**: packages are placed as high as possible
in `node_modules` so they can be shared. If two packages need incompatible versions of
`debug`, one version sits at the top level and the other is nested under the package that
needs it:

```text
node_modules/
  debug/            (4.x, shared by most)
  old-lib/
    node_modules/
      debug/        (2.x, only old-lib sees this one)
```

This works because of Node's module resolution (lesson 03): `old-lib` requiring `debug`
finds its own nested copy first. A side effect: your code can `import` a package you
never declared, because it happens to be hoisted. That is a **phantom dependency** and it
will break when the tree changes. pnpm avoids this with a content-addressed store and
symlinks that expose only declared dependencies.

**The cache and integrity.** Downloaded tarballs go into a local cache
(`~/.npm/_cacache`). The lockfile's `integrity` field is a hash (typically SHA-512) of the
tarball; npm verifies it on install, so a tampered or corrupted download fails loudly.

**Lifecycle scripts.** Packages may define `preinstall`, `install`, and `postinstall`
scripts that run on your machine during `npm install` — often to compile native code.
They are also a supply-chain attack vector. `npm install --ignore-scripts` (or
`ignore-scripts=true` in `.npmrc`) disables them; some packages then need a manual
build step.

**`npx`** first looks for the command in local `node_modules/.bin`, then in the npm cache,
and otherwise offers to download the package temporarily. Double-check names you run
with `npx`; a typo can fetch an unrelated package.

## Common mistakes

- **Not committing the lockfile**, or deleting it to "fix" install problems. You lose
  reproducibility and can silently pick up new versions.
- **Using `npm install` in CI.** Use `npm ci`.
- **Putting build/test tools in `dependencies`.** They bloat production installs.
- **Importing a transitive dependency directly** (phantom dependency). If you import it,
  declare it.
- **Blindly running `npm audit fix --force`.** It can jump major versions and break
  your app. Read what it proposes.

## Exercise

1. Start a project, install `express` and `-D supertest`. Open `package-lock.json` and
   find the `integrity` and `resolved` fields for `express`.
2. Run `npm ls --all | head -40` and find a package that appears at more than one
   version. Use `npm explain` to see why.
3. Add `pretest`, `test`, and a `check` script that chains lint and test. Pass a flag
   through with `--` and verify it reached the underlying command.
4. Delete `node_modules`, run `npm ci`, and compare the time with `npm install`.
