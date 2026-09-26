---
description: "Level 4 of Node.js: profiling, memory leaks, diagnostics, TypeScript, hexagonal architecture, job queues, microservices, containers, and observability."
---

# Level 4 · Master <span class="level-badge">Production</span>

Goal: operate Node services with confidence. The first three levels taught you to build a
correct service; this level is about everything that happens after it ships — finding
why it's slow, why its memory grows, and why a request hung at 3 a.m.; keeping the
codebase changeable as it grows; moving work off the request path; talking to other
services without inheriting their failures; packaging and deploying it; seeing inside it
in production; and replacing it without dropping a single request.

Several lessons here depend on infrastructure that can't run inside a lesson (container
runtimes, Redis, message brokers, tracing backends). Where that's the case, the lesson
says clearly what was run while writing it and what wasn't, and gives you a local
alternative wherever one exists — a Postgres-backed job queue, an in-process Postgres for
tests, a console trace exporter.

## Modules

1. [Performance Profiling & Memory Leaks](01-profiling-memory-leaks.md) — CPU profiles, flame graphs, heap snapshots, and a leak found and measured
2. [Diagnostics in Production](02-diagnostics.md) — diagnostic reports, AsyncLocalStorage, diagnostics_channel, and trace flags
3. [TypeScript for Node](03-typescript-for-node.md) — native type stripping, `tsc` as a checker, and typed Express with Zod
4. [Architecture: Layered & Hexagonal](04-architecture.md) — use cases, ports and adapters, and dependency injection with plain functions
5. [Queues & Background Jobs](05-queues-background-jobs.md) — a runnable `SKIP LOCKED` queue, BullMQ, retries, and idempotent jobs
6. [Microservices Communication](06-microservices-communication.md) — timeouts, retries with jitter, circuit breakers, gRPC, and events
7. [Containerizing & Deploying](07-containerize-deploy.md) — multi-stage Dockerfiles, PID 1 and signals, health checks, and CI
8. [Observability with OpenTelemetry](08-observability.md) — traces across services, RED metrics, and correlating signals
9. [Graceful Shutdown & Reliability](09-graceful-shutdown-reliability.md) — draining on SIGTERM, timeouts everywhere, crash-only design
10. [Capstone — A Production-Ready Service](10-capstone.md) — a bookings service built to a production bar, with a review rubric

## What you need before starting

- Levels 1–3. The capstone assumes every earlier technique.
- Chrome (or another Chromium browser) for DevTools profiling and heap snapshots.
- Optional but recommended: Docker (or Podman) for lessons 07–08 and the capstone.

When you finish this level, you have the skills expected of an engineer who owns Node
services in production — and a capstone repository to prove it.
