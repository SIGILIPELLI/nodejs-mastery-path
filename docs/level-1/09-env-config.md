---
description: "Configure Node apps with environment variables: process.env, --env-file and .env files, a validated fail-fast config module, and secrets hygiene."
---

# 09 · Environment Variables & Configuration

The same code runs on your laptop, in CI, in staging, and in production. What changes is
**configuration**: ports, database URLs, API keys, log levels, feature flags. The widely
adopted convention (from the "twelve-factor app" methodology) is to read configuration
from **environment variables**, so the deployment platform can set them without changing
code or committing secrets.

## Reading the environment

Every Node process has `process.env`, an object of string values inherited from the
parent process (usually your shell):

```bash
PORT=4000 LOG_LEVEL=debug node app.js
```

```js
console.log(process.env.PORT);        // '4000' — always a string
console.log(process.env.MISSING);     // undefined
```

Two properties to remember: **values are always strings** (or `undefined`), and anything
you assign to `process.env` is converted to a string (`process.env.X = 1` stores `'1'`;
`process.env.Y = undefined` stores `'undefined'`).

## .env files without a library

For local development, keeping variables in a `.env` file is convenient. Current Node
releases can load one natively — no `dotenv` package needed:

```text title=".env"
PORT=4000
DATABASE_URL=postgres://app:devpass@localhost:5432/app
LOG_LEVEL=debug
```

```bash
node --env-file=.env app.js
node --env-file-if-exists=.env.local --env-file=.env app.js   # optional extra file
```

Or from code, before anything reads the config: `process.loadEnvFile('.env')`.

Variables already present in the real environment take **precedence** over the file.
That is the behavior you want: production sets real values, and a stray `.env` can't
override them. On a test run, `PORT=5000 node --env-file=.env app.js` used port 5000,
not the file's 4000.

## Worked example: a validated config module

Scattering `process.env.X` reads through your code makes it impossible to see what a
service needs, and a typo gives you `undefined` at 3 a.m. instead of an error at
startup. Centralize and validate:

```js title="config.js"
function required(name) {
  const value = process.env[name];
  if (value === undefined || value === '') {
    throw new Error(`Missing required environment variable ${name}`);
  }
  return value;
}

function int(name, fallback) {
  const raw = process.env[name];
  if (raw === undefined) return fallback;
  const n = Number.parseInt(raw, 10);
  if (Number.isNaN(n)) throw new Error(`${name} must be an integer, got "${raw}"`);
  return n;
}

const env = process.env.NODE_ENV ?? 'development';

export const config = Object.freeze({
  env,
  port: int('PORT', 3000),
  databaseUrl: required('DATABASE_URL'),
  logLevel: process.env.LOG_LEVEL ?? (env === 'production' ? 'info' : 'debug'),
  isProd: env === 'production',
});
```

```js title="app.js"
import { config } from './config.js';
console.log(config);
```

Running it three ways:

```text
$ node --env-file=.env app.js
{
  env: 'development',
  port: 4000,
  databaseUrl: 'postgres://app:devpass@localhost:5432/app',
  logLevel: 'debug',
  isProd: false
}

$ node app.js
Error: Missing required environment variable DATABASE_URL

$ PORT=abc node --env-file=.env app.js
Error: PORT must be an integer, got "abc"
```

The service refuses to start with bad configuration — and says exactly why. This
"fail fast" behavior is far better than starting up and failing on the first request.
In Level 2 you'll replace the hand-written checks with a Zod schema that does the same
job with less code.

## Secrets

- **Never commit `.env`.** Add it to `.gitignore` and commit a `.env.example` listing the
  variable names with dummy values, so new developers know what to set.
- In production, secrets come from the platform: container orchestrator secrets, a cloud
  secrets manager, or your CI/CD system's encrypted variables.
- **Don't log the config object** in production — it contains secrets. Log a redacted
  version.
- Environment variables are visible to anything that can inspect the process (e.g.
  `/proc/<pid>/environ` on Linux for the same user) and are inherited by child
  processes. For highly sensitive values, some teams mount secrets as files instead.

## NODE_ENV

`NODE_ENV=production` is a convention many libraries honor: Express, for example,
enables view caching and less verbose error output. Set it in production. But don't
use `NODE_ENV` for *your own* environment names like `staging` — many tools only
recognize `production` vs not. Use a separate variable (`APP_ENV=staging`) for that.

## How It Actually Works

When the operating system starts a process (`execve` on Unix), it passes three things:
the program path, the argument list (`process.argv`), and the **environment block** — an
array of `KEY=value` strings. A child inherits a copy of its parent's environment by
default. Your shell's `export` adds a variable to the environment it passes to commands
it runs; `PORT=4000 node app.js` adds it just for that one command.

Node exposes that block as `process.env`, a special object whose property reads and
writes go to the real process environment (via `getenv`/`setenv`-like calls), which is
why values are coerced to strings. Changing `process.env` affects only the current
process and children spawned *after* the change — never the parent shell.

`--env-file` is handled by Node before your code runs: it parses the file (`KEY=value`
lines, `#` comments, quoted values, multi-line values in quotes) and sets each variable
that isn't already defined. `util.parseEnv(string)` exposes the same parser if you want
the values as an object without touching `process.env`.

## Common mistakes

- **Treating env values as numbers or booleans.** `process.env.DEBUG === true` is never
  true; `'false'` is a truthy string. Parse explicitly.
- **Reading config at import time in many modules**, before `.env` is loaded, when using
  a loader library. Load env first, then import the config module.
- **Defaults for secrets.** A default `JWT_SECRET='dev-secret'` will eventually reach
  production. Make secrets required.
- **Committing `.env`** or pasting secrets into issue trackers and logs.
- **Mutating config at runtime.** Freeze it; config should be read once at startup.

## Exercise

1. Add `REDIS_URL` (optional), `SESSION_SECRET` (required, at least 32 characters), and
   `CORS_ORIGINS` (a comma-separated list parsed into an array) to the config module.
2. Create `.env.example` and a `.gitignore` entry for `.env`. Confirm with
   `git status` that `.env` is ignored.
3. Write `redact(config)` that replaces any key containing `url`, `secret`, `key`, or
   `password` (case-insensitive) with `'***'`, and log the redacted config at startup.
4. Spawn a child process with `child_process.execFileSync('node', ['-p',
   'process.env.PORT'], { env: { ...process.env, PORT: '9999' } })` and explain why the
   parent's `PORT` is unchanged.
