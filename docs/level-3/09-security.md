---
description: "Secure a Node service: OWASP risks in Node terms, helmet headers, rate limiting, ReDoS, prototype pollution, SSRF, npm audit, and the permission model."
---

# 09 · Security for Node Services

Most security failures in Node backends aren't exotic. They are the same categories that
appear year after year in the **OWASP Top 10** — broken access control, injection,
misconfiguration, vulnerable dependencies — expressed through Node-specific mechanics like
a blocked event loop, a polluted prototype, or a malicious `postinstall` script. This
lesson maps those risks onto concrete defenses, and demonstrates three Node-flavored
attacks so you can recognize them.

## The OWASP categories, in Node terms

| Risk | What it looks like in a Node API | Defense (lesson) |
|------|----------------------------------|------------------|
| Broken access control | `GET /tasks/:id` returns any user's task | Scope every query by owner (L2-10) |
| Injection | SQL built with template strings; `exec()` with user input | Parameterized queries (L2-06), `execFile` with arg arrays (L3-04) |
| Cryptographic failures | Fast hashes for passwords, secrets in code | scrypt/argon2 (L2-07), config from env (L1-09) |
| Insecure design | No rate limit on login, unbounded list endpoints | Rate limiting (below), pagination (L2-03) |
| Security misconfiguration | Stack traces in 500s, permissive CORS, missing headers | Error handler (L2-05), CORS allowlist (L2-02), helmet (below) |
| Vulnerable components | Outdated packages with known CVEs | `npm audit`, lockfiles, updates (below) |
| Auth failures | Long-lived JWTs, session fixation | Short tokens, `regenerate()` (L2-07) |
| Integrity failures | Compromised dependency, malicious install script | Lockfile integrity, `--ignore-scripts`, provenance (below) |
| Logging failures | No logs of auth failures; tokens in logs | Pino with redaction (L2-09) |
| SSRF | Server fetches a user-supplied URL | URL allowlists (below) |

## Worked example: headers and rate limiting

```js title="security-basics.js"
import express from 'express';
import helmet from 'helmet';
import { rateLimit } from 'express-rate-limit';
import request from 'supertest';

const app = express();
app.use(helmet());
app.use('/auth/login', rateLimit({ windowMs: 60_000, limit: 5, standardHeaders: 'draft-8', legacyHeaders: false }));
app.post('/auth/login', (req, res) => res.status(401).json({ error: 'invalid email or password' }));

const r = await request(app).post('/auth/login');
for (const h of ['content-security-policy', 'strict-transport-security', 'x-content-type-options', 'x-frame-options', 'x-powered-by', 'ratelimit', 'ratelimit-policy']) {
  console.log(h.padEnd(26), r.headers[h] ?? '(absent)');
}
const codes = [];
for (let i = 0; i < 6; i++) codes.push((await request(app).post('/auth/login')).status);
console.log('next 6 attempts:', codes.join(' '));
```

```text
content-security-policy    default-src 'self';base-uri 'self';font-src 'self' https: data:;form-action 'self';frame-ancestors 'self';img-src 'self' data:;object-src 'none';script-src 'self';script-src-attr 'none';style-src 'self' https: 'unsafe-inline';upgrade-insecure-requests
strict-transport-security  max-age=31536000; includeSubDomains
x-content-type-options     nosniff
x-frame-options            SAMEORIGIN
x-powered-by               (absent)
ratelimit                  "5-in-1min"; r=4; t=60
ratelimit-policy           "5-in-1min"; q=5; w=60; pk=:MTJjYTE3YjQ5YWYy:
next 6 attempts: 401 401 401 401 429 429
```

- **helmet** sets a batch of protective response headers (Content-Security-Policy,
  HSTS, `nosniff`, frame restrictions) and removes `X-Powered-By`. They matter most when
  your server also returns HTML; they're cheap to have on APIs.
- **express-rate-limit** allowed five attempts per minute (the first request plus four
  more) and then answered 429, advertising its policy in standard `RateLimit` headers.
  Its default store is in memory — per process — so with multiple instances use a Redis
  store. Behind a proxy or load balancer, configure Express's `trust proxy` setting so the
  limiter sees the client's IP rather than the proxy's.

## Three Node-specific attacks, demonstrated

### 1. ReDoS: regular-expression denial of service

JavaScript's regex engine backtracks. Some patterns take exponential time on crafted
input, and because matching runs on the main thread, **one request freezes the whole
process**:

```js title="redos.mjs"
// A classic catastrophic-backtracking pattern: nested quantifiers
const evil = /^(a+)+$/;
for (const n of [20, 24, 26, 28]) {
  const input = 'a'.repeat(n) + '!';
  const t0 = performance.now();
  evil.test(input);
  console.log(`n=${n}: ${Math.round(performance.now() - t0)} ms`);
}
```

```text
n=20: 73 ms
n=24: 105 ms
n=26: 429 ms
n=28: 1688 ms
```

Each extra couple of characters roughly quadruples the time. A 40-character input would
block for far longer than any request timeout. Defenses: avoid nested quantifiers like
`(a+)+` and `(.*a){n}`, cap input length **before** matching, prefer purpose-built
validators (Zod's `z.email()`) over hand-rolled regexes, and use lint rules that flag
unsafe patterns.

### 2. Prototype pollution

```js title="proto.mjs"
// Naive deep merge, as found in many hand-written config/utility helpers
function merge(target, source) {
  for (const key of Object.keys(source)) {
    if (typeof source[key] === 'object' && source[key] !== null) {
      target[key] ??= {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

const userInput = JSON.parse('{"theme": "dark", "__proto__": {"isAdmin": true}}');
merge({}, userInput);
const someoneElse = {};
console.log('someoneElse.isAdmin =', someoneElse.isAdmin);
```

```text
someoneElse.isAdmin = true
```

`JSON.parse` creates an own property literally named `__proto__`. The naive merge then
does `target['__proto__']`, which is `Object.prototype`, and copies `isAdmin` onto it —
so **every object in the process** now has `isAdmin === true`. If any code checks
`if (user.isAdmin)` on an object without that own property, the attacker is an admin.
Defenses: validate input with a schema that strips unknown keys (Level 2, lesson 04);
skip `__proto__`, `constructor`, and `prototype` keys in merge utilities; use
`Object.create(null)` or `Map` for dictionaries keyed by user input; keep dependencies
patched (several popular utility libraries have had pollution CVEs).

### 3. SSRF: server-side request forgery

Any feature where your server fetches a user-supplied URL — webhooks, link previews,
"import from URL" — can be aimed at internal addresses: `http://localhost:6379`,
`http://10.0.0.5/admin`, or a cloud provider's instance-metadata endpoint that hands out
credentials. Defenses: allowlist destinations where possible; otherwise resolve the
hostname, reject private, loopback, and link-local ranges, and **connect to the IP you
validated** (re-resolving later allows DNS-rebinding tricks); disable redirects or
re-validate each hop; and run such fetches from an egress-restricted network.

## Dependencies and the supply chain

A typical Node API has hundreds of transitive dependencies, any of which runs with your
process's full privileges.

- **`npm audit`** compares your lockfile against a vulnerability database and reports
  known issues with fix suggestions (our scratch project for this course reported
  `found 0 vulnerabilities` at the time of writing — that changes as advisories are
  published). Run it in CI; triage rather than blindly `--force`-fixing.
- **Keep the lockfile** and install with `npm ci` so builds use exactly the audited tree.
- **Automated update PRs** (Dependabot, Renovate) keep patches flowing in small, testable
  steps.
- **Install scripts** (`preinstall`/`postinstall`) execute on `npm install`. Consider
  `ignore-scripts=true` in `.npmrc` and allowlisting packages that genuinely need a build.
- **Fewer dependencies** is a security feature. Node now covers a lot natively: `fetch`,
  `node:test`, `--env-file`, `crypto.randomUUID`, `util.parseArgs`, `WebSocket` client.
- **Provenance**: npm supports publishing packages with signed build provenance; `npm
  audit signatures` verifies registry signatures and provenance attestations of installed
  packages.

## The Node permission model

Node has an opt-in **permission model** that restricts what a process may do:

```bash
node --permission --allow-fs-read="$PWD" app.mjs
```

With that flag, an attempt to read `/etc/hosts` from outside the allowed directory threw:

```text
Error: Access to this API has been restricted. Use --allow-fs-read to manage permissions.
  code: 'ERR_ACCESS_DENIED',
  permission: 'FileSystemRead',
```

Flags exist for file-system reads and writes, child processes, worker threads, and native
addons. Treat it as defense in depth (it constrains a compromised dependency) rather than a
sandbox for running untrusted code; container isolation and least-privilege OS users
remain the primary boundary.

## How It Actually Works

**Why ReDoS hurts Node especially:** V8's default regex engine (Irregexp) is a
backtracking engine. For `^(a+)+$` on `aaaa…!`, when the final `$` fails, the engine
tries every way of splitting the run of `a`s between the inner and outer `+` — 2^(n-1)
ways. In a thread-per-request server this ties up one thread; in Node it ties up *the*
thread. V8 also ships an experimental linear-time engine, enabled with flags, that trades
features (no backreferences) for guaranteed linear time.

**Why prototype pollution works:** property lookup walks the prototype chain. Plain
objects inherit from `Object.prototype`, so a property added there is visible on every
object that doesn't shadow it. The assignment `obj['__proto__'] = value` doesn't create
a property — it invokes the `__proto__` accessor, which either returns or replaces the
prototype, so recursing into it reaches the shared prototype itself.

**How rate limiting counts:** the limiter derives a key (default: client IP), increments
a counter in its store for the current window, and rejects once the counter exceeds the
limit. With the memory store the counter lives in a `Map` in this process — correct only
when one process handles all traffic for that key.

## Common mistakes

- **Security middleware registered after routes**, so it never runs for them.
- **Rate limiting by IP behind a proxy without `trust proxy`** → everyone shares one limit
  (the proxy's IP).
- **Validating output of `JSON.parse` with `if (obj.isAdmin)`**-style checks on objects
  that could be polluted.
- **`npm audit fix --force`** without reading what changes.
- **Secrets in the repository** or in Docker image layers.
- **Returning detailed error messages** that reveal internals (SQL, paths, versions).

## Exercise

1. Add helmet and a login rate limiter to the Level 2 project; write a test for the 429.
2. Fix the naive `merge` to be pollution-safe, and write a test that proves
   `({}).isAdmin` stays `undefined` after merging the malicious payload.
3. Find a regex in a project you've written (or the Level 1 log parser). Test it with long
   adversarial inputs and cap the input length before matching.
4. Write `safeFetch(url)` for a link-preview feature that only allows `https:`, resolves
   the hostname with `dns.lookup(host, { all: true })`, rejects private/loopback ranges,
   and disables redirects. List what it still doesn't protect against.
