---
description: "Observability for Node services: logs, metrics and traces, OpenTelemetry setup for ESM, context propagation across services, RED metrics, and alerts."
---

# 08 · Observability with OpenTelemetry

Monitoring tells you *that* something is wrong; **observability** lets you ask *why*
without shipping new code. It rests on three kinds of telemetry:

- **Logs** — discrete events with context (Level 2, lesson 09).
- **Metrics** — numbers aggregated over time: request rate, error rate, latency
  percentiles, event-loop delay, heap size, queue depth. Cheap to store, ideal for
  dashboards and alerts.
- **Traces** — the path of one request through your services, as a tree of timed
  **spans**. They answer "which of the 14 calls behind this slow request was slow?".

**OpenTelemetry (OTel)** is the vendor-neutral standard for producing all three. You
instrument once with OTel's API and SDK, export over the OTLP protocol to a **collector**,
and route data to whichever backend you use — Jaeger, Grafana Tempo/Prometheus/Loki,
Honeycomb, Datadog, New Relic, a cloud provider's tracing service, and so on.

## Setting up tracing in Node

```bash
npm install @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/sdk-trace-node \
  @opentelemetry/instrumentation-http @opentelemetry/resources @opentelemetry/semantic-conventions
```

Instrumentation must load **before** your application imports the modules it patches, so
it lives in its own file loaded with `--import`:

```js title="instrumentation.mjs"
import { register } from 'node:module';
// ESM needs a loader hook so instrumentations can patch modules loaded with `import`
register('@opentelemetry/instrumentation/hook.mjs', import.meta.url);

import { NodeSDK } from '@opentelemetry/sdk-node';
import { ConsoleSpanExporter, SimpleSpanProcessor } from '@opentelemetry/sdk-trace-node';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { resourceFromAttributes } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: resourceFromAttributes({ [ATTR_SERVICE_NAME]: 'orders-api' }),
  // Console exporter for learning; production uses an OTLP exporter to a collector
  spanProcessors: [new SimpleSpanProcessor(new ConsoleSpanExporter())],
  instrumentations: [new HttpInstrumentation()],
});
sdk.start();
process.on('SIGTERM', () => sdk.shutdown().finally(() => process.exit(0)));
```

The `register(...)` line matters for ES modules. Instrumentations work by patching modules
as they load. For CommonJS `require` that's easy to intercept; for ESM `import`, a module
loader hook is needed. The first version of this lesson's demo omitted it — and only the
manually created span appeared; the HTTP server and client spans were silently missing.

For broad coverage (Express, pg, Redis clients, undici/fetch, and more), the
`@opentelemetry/auto-instrumentations-node` package bundles many instrumentations.

## Worked example: a trace across two services

```js title="app.mjs"
import { createServer, get } from 'node:http';
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('orders');

// "inventory" service on 4001
createServer((req, res) => res.end('{"inStock":3}')).listen(4001);

// "orders" service on 4000: calls inventory, and records a custom span for business logic
createServer(async (req, res) => {
  const stock = await new Promise((resolve) => get('http://localhost:4001/stock/LAMP', (r) => {
    let body = ''; r.on('data', (c) => (body += c)); r.on('end', () => resolve(JSON.parse(body)));
  }));
  await tracer.startActiveSpan('price-order', async (span) => {
    span.setAttribute('order.items', 2);
    span.setAttribute('inventory.in_stock', stock.inStock);
    span.setStatus({ code: SpanStatusCode.OK });
    span.end();
  });
  res.end('ok');
}).listen(4000, () => {
  get('http://localhost:4000/orders', (r) => r.resume().on('end', () => setTimeout(() => process.exit(0), 200)));
});
```

```bash
node --import ./instrumentation.mjs app.mjs
```

The console exporter printed five spans sharing one `traceId`. Reassembled by their
parent ids, they form this tree (ids from the actual run):

```text
trace efc58a5d5eea0b5d52282dbe5674fab8
└─ GET (client)   a109b7c9…  url.full=http://localhost:4000/orders          status 200
   └─ GET (server) 727e68db…  orders service
      ├─ GET (client)  f52f6aa6…  url.full=http://localhost:4001/stock/LAMP  status 200
      │  └─ GET (server) aebd9e71…  inventory service
      └─ price-order   9b414b5d…  order.items=2, inventory.in_stock=3
```

(`kind` in the raw output: 1 = server, 2 = client, 0 = internal.) In a tracing UI this is
a waterfall showing each span's duration. Two things happened automatically:

1. The HTTP client instrumentation **injected a `traceparent` header** into the call to
   inventory, and the server instrumentation **extracted** it — so the inventory span
   became a child in the same trace. That's **context propagation**; it works across
   languages because `traceparent` is a W3C standard.
2. The custom `price-order` span became a child of the current server span without
   passing anything explicitly, thanks to AsyncLocalStorage-based context (lesson 02).

Add custom spans for meaningful business steps and attributes you'd want to filter by
(tenant id, order size, cache hit/miss) — but never secrets or raw personal data.

## Metrics that matter for a Node service

The **RED** method for every service endpoint:

- **Rate** — requests per second,
- **Errors** — failed requests per second (5xx, and timeouts),
- **Duration** — latency distribution (p50, p95, p99 — not averages).

Plus Node runtime health:

- **Event-loop delay** p99 (L3-01) — the most Node-specific signal of trouble.
- **Heap used after GC** and **RSS** (lesson 01), GC pause time.
- **Pool saturation** — DB pool waiting count, libuv thread-pool pressure (indirectly via
  fs/dns latency).
- **Queue depth and oldest job age** for workers (lesson 05).

OTel's metrics API (`meter.createHistogram('http.server.request.duration')`) exports to
OTLP; `prom-client` is a widely used alternative that exposes a `/metrics` endpoint in
Prometheus format and includes default Node runtime metrics.

## Correlating the three signals

Put the trace id in every log line (OTel's Pino instrumentation does this automatically,
or read it via `trace.getActiveSpan()?.spanContext().traceId`). Then a spike in the error
metric → pick an example trace → see the failing span → jump to that trace's logs.
Without correlation you're grepping timestamps.

## Alerting

Alert on **symptoms users feel**, based on your SLOs: error rate above threshold, p99
latency above threshold, for several minutes. Use cause-level metrics (heap growth, event
loop delay, pool saturation) for dashboards and diagnosis, and alert on them only when they
reliably predict user pain. Every alert should have a runbook: what it means, how to check,
how to mitigate.

## How It Actually Works

**Spans** are records with a trace id (16 bytes), span id (8 bytes), parent span id, name,
kind, start/end timestamps, attributes, events, and status. The SDK keeps the **active
span** in the current context (backed by AsyncLocalStorage), so any span started while
another is active becomes its child.

**Propagation** serializes the active context into outgoing headers. The W3C
`traceparent` header looks like `00-<trace-id>-<parent-span-id>-<flags>`; the receiving
service parses it and creates its server span with that parent. Baggage (`baggage`
header) can carry extra key-values across services.

**Instrumentation** libraries wrap functions of the target module (`http.request`,
`Server.prototype.emit('request')`, `pg.Client.query`): start a span before, end it after,
record attributes and errors. That's why they must load first, and why ESM needs a loader
hook — without one, the application gets the original, unwrapped module.

**Export**: `SimpleSpanProcessor` exports each span as it ends (fine for demos). Production
uses `BatchSpanProcessor`, which buffers spans and exports in the background on a timer,
dropping spans rather than blocking your app if the buffer fills. Most setups also
**sample** (e.g. keep 10% of traces, or all traces with errors) to control cost; the sampling
decision is made at the root and propagated in the `traceparent` flags so every service
keeps or drops the same traces.

## Common mistakes

- **Loading instrumentation after the app** (or forgetting the ESM hook) → no automatic spans.
- **High-cardinality metric labels** (user ids, raw URLs) → exploding storage costs. Use
  route templates (`/tasks/:id`), not paths.
- **Averages instead of percentiles** for latency.
- **Secrets or personal data in span attributes.**
- **Not flushing on shutdown** — call `sdk.shutdown()` so buffered spans are exported.
- **Alerting on everything**, so real alerts get ignored.

## Exercise

1. Add OTel to the Level 2 tasks API with auto-instrumentations for http, express, and pg.
   Run a local Jaeger (`docker run -p 16686:16686 -p 4318:4318 jaegertracing/all-in-one`,
   or any OTLP-compatible backend) and switch to the OTLP HTTP exporter.
2. Add a custom span around password hashing with an attribute for the cost parameter, and
   find it in a login trace.
3. Expose a `/metrics` endpoint with `prom-client`'s default metrics plus an HTTP duration
   histogram labelled by method, route template, and status class.
4. Log the trace id in every Pino line and demonstrate going from a log line to its trace.
