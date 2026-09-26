---
description: "Containerize and deploy a Node service: a multi-stage Dockerfile, npm ci, non-root users, signals as PID 1, health checks, memory limits, and CI."
---

# 07 · Containerizing & Deploying

A service isn't finished when it works on your laptop; it's finished when it runs
reliably somewhere else, is rebuilt identically every time, and can be replaced by a new
version without dropping requests. Containers are the most common packaging for Node
services today, whether they end up on Kubernetes, a managed container platform (Cloud
Run, ECS/Fargate, Azure Container Apps, Fly.io, Render), or a single VM with Docker
Compose.

This lesson's files were written for Docker but **not built or run while writing it** —
the authoring machine had no container runtime. Every instruction used is standard
Dockerfile syntax; build it yourself and fix anything your environment disagrees with.

## A production Dockerfile

```dockerfile title="Dockerfile"
# syntax=docker/dockerfile:1

# ---- deps: install production dependencies from the lockfile ----
FROM node:22-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ---- build: only needed if you compile (TypeScript to JS, bundling, etc.) ----
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run typecheck && npm test

# ---- runtime: small, non-root, no build tools ----
FROM node:22-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --chown=node:node package.json ./
COPY --chown=node:node src ./src
USER node
EXPOSE 3000
CMD ["node", "src/server.js"]
```

Decisions, line by line:

- **Pin a major Node version** that matches your `.nvmrc` (use your current LTS; `22` is
  just an example). For fully reproducible builds, pin the image digest
  (`node:22-slim@sha256:...`) and let a bot update it.
- **`-slim`** (Debian-based, without compilers) is a good default. `-alpine` is smaller but
  uses musl libc, which some native modules don't support well. "Distroless" images are
  smaller still, with no shell.
- **Copy the manifests before the source**, so the `npm ci` layer is cached and only
  re-runs when dependencies change.
- **`npm ci --omit=dev`** installs exactly the lockfile, without test tools.
- The **build stage runs typecheck and tests**, so a broken build can't produce an image.
  (Some teams run these in CI instead; don't skip them in both.)
- **`USER node`** — the official images include an unprivileged `node` user. A compromised
  process shouldn't be root inside the container.
- **`CMD ["node", ...]` in exec form**, not `npm start` and not the shell form — see "How
  It Actually Works" for why this matters for shutdown.

And keep junk out of the build context:

```text title=".dockerignore"
node_modules
.git
.env
*.log
coverage
Dockerfile
.dockerignore
```

Excluding `.env` matters: without it, `COPY . .` bakes your local secrets into an image
layer, where anyone who can pull the image can read them.

## Build and run

```bash
docker build -t tasks-api:dev .
docker run --rm -p 3000:3000 \
  -e DATABASE_URL=postgres://tasks:devpass@host.docker.internal:5432/tasks \
  -e JWT_SECRET="$(openssl rand -hex 32)" \
  --memory=512m \
  tasks-api:dev
```

For local development with dependencies, a `compose.yaml` runs the API, Postgres, and
Redis together:

```yaml title="compose.yaml"
services:
  api:
    build: .
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://tasks:devpass@db:5432/tasks
      JWT_SECRET: dev-only-secret-at-least-32-characters-long
    depends_on:
      db: { condition: service_healthy }
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: tasks
      POSTGRES_PASSWORD: devpass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tasks"]
      interval: 2s
      retries: 20
```

## Health checks

Platforms need to know whether an instance should receive traffic. Expose two endpoints:

- **Liveness** (`/livez`): "the process is not wedged" — return 200 if the event loop
  responds. Failing it gets the container restarted. Don't check dependencies here, or a
  database blip restarts every instance at once.
- **Readiness** (`/readyz`): "I can serve traffic now" — check that the DB pool can run
  `select 1` and that you're not shutting down. Failing it removes the instance from the
  load balancer without restarting it.

```js
let shuttingDown = false;
app.get('/livez', (req, res) => res.json({ ok: true }));
app.get('/readyz', async (req, res) => {
  if (shuttingDown) return res.status(503).json({ ok: false, reason: 'shutting down' });
  try {
    await sql`select 1`.execute(db);
    res.json({ ok: true });
  } catch {
    res.status(503).json({ ok: false, reason: 'database unavailable' });
  }
});
```

## Memory limits

Containers have a hard memory limit; exceeding it gets the process killed by the kernel
(OOM kill, exit code 137) with no JavaScript error at all. Recent Node versions size the
V8 heap from the container's cgroup limit, but the heap isn't all of your memory (buffers,
native code, thread stacks). Leave headroom: for a 512 MB container, a heap of around
300–400 MB is a common starting point. Setting it explicitly makes behavior predictable:
`NODE_OPTIONS=--max-old-space-size=384`. Watch `rss` in your metrics to calibrate.

## A CI pipeline

```yaml title=".github/workflows/ci.yml"
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env: { POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
        options: --health-cmd "pg_isready" --health-interval 2s --health-retries 20
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc, cache: npm }
      - run: npm ci
      - run: npm run typecheck --if-present
      - run: npm test
        env: { DATABASE_URL: "postgres://postgres:test@localhost:5432/postgres" }
      - run: npm audit --audit-level=high
  image:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t tasks-api:${{ github.sha }} .
      # push to your registry and deploy — details depend on your platform
```

Deploys should be **rolling**: start new instances, wait for readiness, shift traffic,
then stop old instances gracefully (lesson 09). Database migrations run as a separate step
before the new version takes traffic, and must be backward compatible with the version
still running (add columns first, remove them in a later release).

## How It Actually Works

**Image layers.** Each Dockerfile instruction produces a layer: a filesystem diff stored
by content hash. On rebuild, Docker reuses cached layers until the first instruction whose
inputs changed; everything after it is rebuilt. That's why copying `package*.json` and
running `npm ci` *before* `COPY src` keeps rebuilds fast. Multi-stage builds let the final
image copy only chosen files from earlier stages, leaving compilers, dev dependencies, and
source-only files behind.

**PID 1 and signals.** Inside a container, your command runs as **PID 1**. To stop a
container, the runtime sends `SIGTERM` to PID 1, waits (10 seconds by default in Docker,
30 in Kubernetes), then sends `SIGKILL`. Two traps:

1. With `CMD npm start` (or the shell form `CMD node src/server.js`), PID 1 is `npm` or
   `/bin/sh`, and depending on versions it may not forward `SIGTERM` to Node — your
   graceful shutdown code never runs and the process is killed after the timeout.
2. The kernel treats PID 1 specially: signals with no handler installed are **ignored**
   for PID 1 instead of using the default action (terminate). Node doesn't install its own
   `SIGTERM` handler, so if your app registers none and runs as PID 1, it may simply
   ignore `SIGTERM` until it's killed. Handle the signal (lesson
   09), or run with an init process (`docker run --init`, or `tini`) that forwards signals
   and reaps zombie processes.

**Cgroup memory limits.** The container's limit is enforced by the Linux kernel's cgroup
controller. When the cgroup exceeds it, the kernel's OOM killer terminates a process in it
— from Node's point of view, instantly and without warning.

## Common mistakes

- **Secrets in images** (`COPY .env`, `ENV API_KEY=...`, or `ARG` values visible in history).
- **`npm install` instead of `npm ci`**, producing unreproducible images.
- **Running as root.**
- **`CMD npm start`** and then wondering why shutdown isn't graceful.
- **Liveness probes that check the database**, causing cascading restarts.
- **No memory limit or heap sizing**, leading to OOM kills that look like random restarts.
- **Dev dependencies and source maps of secrets** shipped in the runtime image.

## Exercise

1. Write a Dockerfile for the Level 2 tasks API, build it, and compare the image size with
   `node:22` (full) vs `node:22-slim` as the runtime base.
2. Run the container, then `docker stop` it and time how long it takes. Add a `SIGTERM`
   handler (lesson 09) and time it again.
3. Add `/livez` and `/readyz` to the tasks API; stop Postgres and confirm readiness fails
   while liveness stays healthy.
4. Run the container with `--memory=128m` and a leaky endpoint; observe the exit code
   (`docker inspect --format '{{.State.ExitCode}} {{.State.OOMKilled}}'`).
