---
description: "Authentication for Node APIs: hashing passwords with scrypt, server-side sessions vs JWTs, secure cookies, bearer-token middleware, and common auth flaws."
---

# 07 · Authentication: Sessions, JWT & Password Hashing

Authentication answers "who is making this request?". For a typical API that means two
separate problems:

1. **Verifying a password once**, at login — which requires storing passwords safely.
2. **Recognizing the same user on later requests** without asking for the password each
   time — via a **session** or a **token**.

Authorization ("is this user allowed to do *that*?") comes after, and is usually plain
code: `where user_id = req.user.id`, role checks, ownership checks.

## Part 1: storing passwords

Never store passwords, and never store a plain fast hash like SHA-256 of a password.
Password databases leak; attackers then guess billions of candidate passwords per second
against fast hashes on GPUs. Use a **deliberately slow, salted, memory-hard** password
hashing function: **argon2id**, **scrypt**, or **bcrypt**. Node has scrypt built in
(`node:crypto`), so no native dependency is needed. (The `argon2` and `bcrypt` npm
packages are good choices too; they compile native code.)

```js title="src/auth/password.js"
import { scrypt, randomBytes, timingSafeEqual } from 'node:crypto';
import { promisify } from 'node:util';

const scryptAsync = promisify(scrypt);
const KEYLEN = 64;
const PARAMS = { N: 16384, r: 8, p: 1 }; // cost parameters, stored with each hash

export async function hashPassword(password) {
  const salt = randomBytes(16);
  const key = await scryptAsync(password, salt, KEYLEN, PARAMS);
  return `scrypt$${PARAMS.N}$${PARAMS.r}$${PARAMS.p}$${salt.toString('base64')}$${key.toString('base64')}`;
}

export async function verifyPassword(password, stored) {
  const [algo, N, r, p, saltB64, keyB64] = stored.split('$');
  if (algo !== 'scrypt') return false;
  const expected = Buffer.from(keyB64, 'base64');
  const actual = await scryptAsync(password, Buffer.from(saltB64, 'base64'), expected.length,
    { N: Number(N), r: Number(r), p: Number(p) });
  return timingSafeEqual(actual, expected);
}
```

Observed behavior:

```js
const stored = await hashPassword('correct horse battery');
console.log(stored.slice(0, 40) + '...');
await verifyPassword('correct horse battery', stored);   // true
await verifyPassword('Correct horse battery', stored);   // false
(await hashPassword('correct horse battery')) === stored; // false — new random salt each time
```

```text
scrypt$16384$8$1$3QS+jXtaX53F9nIFKUzmpQ=...
hash took 36 ms
true false
false
```

Design points:

- A random **salt** per password means identical passwords produce different hashes, so
  attackers can't use precomputed tables or crack all users with one guess.
- The **parameters are stored with the hash**. You can raise the cost later; old hashes
  still verify with their own parameters, and you rehash on next successful login.
- `timingSafeEqual` compares in constant time so response timing doesn't leak how many
  bytes matched.
- A few tens of milliseconds per hash is the point: negligible for one login, ruinous
  for an attacker trying billions. Tune `N` upward as far as your login latency budget
  and memory allow.

## Part 2a: server-side sessions

The server creates a random session id, stores session data (e.g. `userId`) in a store,
and sends the id in a cookie. The browser returns the cookie automatically on every
request.

```bash
npm install express-session
```

```js title="sessions.js"
import express from 'express';
import session from 'express-session';

const app = express();
app.use(express.json());
app.use(session({
  name: 'sid',
  secret: process.env.SESSION_SECRET ?? 'dev-only-secret-change-me-32-chars!!',
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, sameSite: 'lax', secure: process.env.NODE_ENV === 'production', maxAge: 1000 * 60 * 60 * 8 },
  // store: new RedisStore({ client }),  // production: a shared store, not memory
}));

app.post('/login', (req, res, next) => {
  // ... verify email + password as in the JWT version ...
  const user = { id: 42, email: req.body.email };
  req.session.regenerate((err) => {          // new session id on login: prevents fixation
    if (err) return next(err);
    req.session.userId = user.id;
    res.json({ ok: true });
  });
});

app.get('/me', (req, res) => {
  if (!req.session.userId) return res.status(401).json({ error: 'not logged in' });
  res.json({ userId: req.session.userId });
});

app.post('/logout', (req, res, next) => {
  req.session.destroy((err) => {
    if (err) return next(err);
    res.clearCookie('sid').status(204).end();
  });
});
```

Driving it with Supertest's `agent` (which keeps cookies like a browser):

```text
sid=<signed id>; Path=/; Expires=...; HttpOnly; SameSite=Lax
{ userId: 42 }
401        ← GET /me after logout
```

Cookie flags matter as much as the session itself:

- `HttpOnly` — JavaScript in the page can't read the cookie, so an XSS bug can't simply
  steal it.
- `Secure` — only sent over HTTPS (enable in production).
- `SameSite=Lax` — not sent on cross-site POSTs, which blocks most CSRF. For extra
  protection on state-changing requests, add CSRF tokens or require a custom header.
- `regenerate()` on login issues a fresh id, preventing **session fixation**.

The default `MemoryStore` is for development only: it leaks memory and isn't shared
across processes. Use Redis or your database as the store in production.

## Part 2b: JSON Web Tokens

A JWT is a signed, self-contained token: `header.payload.signature`, each part
base64url-encoded. The server signs it at login; the client sends it back in an
`Authorization: Bearer <token>` header; the server verifies the signature and trusts the
payload — **no lookup needed**.

```bash
npm install jsonwebtoken
```

```js title="src/auth/tokens.js"
import jwt from 'jsonwebtoken';
import { unauthorized } from '../errors.js';

export function signToken(user, secret) {
  return jwt.sign({ sub: String(user.id), email: user.email }, secret, {
    algorithm: 'HS256',
    expiresIn: '15m',
  });
}

export function requireAuth(secret) {
  return (req, res, next) => {
    const header = req.get('authorization') ?? '';
    const [scheme, token] = header.split(' ');
    if (scheme !== 'Bearer' || !token) return next(unauthorized());
    try {
      const payload = jwt.verify(token, secret, { algorithms: ['HS256'] });
      req.user = { id: Number(payload.sub), email: payload.email };
      next();
    } catch {
      next(unauthorized('invalid or expired token'));
    }
  };
}
```

Decoding a token shows the payload is **readable by anyone** — it's encoded, not
encrypted:

```text
{ alg: 'HS256', typ: 'JWT' } { sub: '42', role: 'member', iat: 1790435096, exp: 1790435996 }
```

Changing `role` to `admin` in the payload and re-encoding it fails verification:

```text
JsonWebTokenError invalid signature
```

Never put secrets in a JWT payload, and always pass an explicit `algorithms` list to
`verify` so an attacker can't choose the algorithm for you.

## Sessions vs JWT: choosing

| | Server-side session | JWT (stateless) |
|--|--|--|
| Revocation (logout, ban) | Delete the session → immediate | Hard: token valid until `exp` unless you keep a denylist |
| Per-request cost | A store lookup (fast with Redis) | Signature check only |
| Size | Small cookie | Larger header, grows with claims |
| Browser apps | Natural fit with `HttpOnly` cookies | Storing in `localStorage` exposes it to XSS |
| Service-to-service, mobile, third-party APIs | Awkward | Natural fit |

A reasonable default: **cookie sessions for your own browser frontend**, **short-lived
JWTs (minutes) plus a revocable refresh token** for mobile clients and APIs. The Level 2
project uses 15-minute bearer JWTs to keep the example small; production systems add a
refresh-token flow stored server-side.

## How It Actually Works

**scrypt** derives a key by filling a large memory buffer (roughly `128 * N * r` bytes —
16 MiB with the parameters above) with pseudo-random data derived from the password and
salt, then reading it back in a data-dependent order. An attacker's GPU or ASIC must
provide that memory for every parallel guess, which is what makes it expensive to
attack at scale. In Node, `crypto.scrypt` runs on the **libuv thread pool**, so it
doesn't block the event loop — but with four pool threads, a burst of logins can queue up
and delay unrelated `fs` or DNS work. Rate-limit your login endpoint.

**HS256 JWT signing** is HMAC-SHA256 over the ASCII string
`base64url(header) + "." + base64url(payload)` with your secret as the key. Verification
recomputes the HMAC and compares; any change to header or payload changes the MAC. With
**RS256/ES256**, a private key signs and a public key verifies, so other services can
verify tokens without being able to mint them.

**Signed session cookies** in `express-session` contain the session id plus an HMAC
(`s:<id>.<signature>`), so a client can't forge ids. The id itself is random and
unguessable; the data lives server-side.

## Common mistakes

- **Fast hashes (MD5, SHA-*) or reversible encryption for passwords.**
- **Different errors for "no such user" and "wrong password"** — lets attackers enumerate
  accounts. The project returns the same 401 for both.
- **Long-lived JWTs with no revocation strategy.**
- **Weak or committed JWT secrets.** Use at least 32 random bytes from config.
- **Not specifying `algorithms` in `jwt.verify`.**
- **Storing tokens in `localStorage`** for browser apps.
- **No rate limiting on login** — enables brute-force and thread-pool exhaustion.

## Exercise

1. Add `POST /auth/change-password` that requires the current password, rehashes, and
   (for the session version) regenerates the session.
2. Implement "rehash on login": if a stored hash's `N` is lower than the current
   parameters, compute a new hash after successful verification and update the row.
3. Add refresh tokens: a random 32-byte token stored hashed in a `refresh_tokens` table
   with an expiry, exchanged at `POST /auth/refresh` for a new access JWT, and deleted on
   logout.
4. Use `express-rate-limit` to allow at most 5 login attempts per minute per IP, and
   write a test proving the sixth gets a 429.
