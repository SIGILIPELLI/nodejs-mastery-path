---
description: "Master Node's EventEmitter: on, once, emit, the special error event, synchronous dispatch, events.once and async iteration, and listener leaks."
---

# 07 · Events & EventEmitter

Much of Node's core is built on one small class: `EventEmitter`. HTTP servers emit
`'request'`, sockets emit `'data'` and `'close'`, streams emit `'end'`, the `process`
object emits `'exit'` and `'SIGTERM'`. Once you understand how an emitter dispatches
events, a large part of the standard library stops being mysterious.

## The API in five methods

```js
import { EventEmitter } from 'node:events';

const bus = new EventEmitter();
bus.on('tick', (n) => console.log('tick', n));     // add listener
bus.once('tick', () => console.log('first tick')); // listener removed after one call
bus.emit('tick', 1);                                // call listeners with args
bus.off('tick', someFn);                            // remove a specific listener
bus.listenerCount('tick');                          // how many are attached
```

`emit` returns `true` if there were listeners, `false` otherwise.

## Worked example: an order queue

Subclassing `EventEmitter` is the usual way to give your own objects events:

```js title="events.mjs"
import { EventEmitter, once } from 'node:events';

class OrderQueue extends EventEmitter {
  #orders = [];
  add(order) {
    if (!order.id) {
      this.emit('error', new Error('order without id'));
      return;
    }
    this.#orders.push(order);
    this.emit('added', order, this.#orders.length);
    if (this.#orders.length >= 3) this.emit('full');
  }
}

const q = new OrderQueue();
q.on('added', (order, size) => console.log(`added #${order.id} (size ${size})`));
q.once('full', () => console.log('queue full - flushing'));
q.on('error', (err) => console.log('handled error:', err.message));

console.log('before add');
q.add({ id: 1 });
console.log('after add');
q.add({});
q.add({ id: 2 });
q.add({ id: 3 });
q.add({ id: 4 });

setTimeout(() => q.emit('ready', 'db'), 50);
const [what] = await once(q, 'ready');
console.log('ready:', what, '| listeners for added:', q.listenerCount('added'));
```

```text
before add
added #1 (size 1)
after add
handled error: order without id
added #2 (size 2)
added #3 (size 3)
queue full - flushing
added #4 (size 4)
ready: db | listeners for added: 1
```

Notice three things:

1. `added #1` printed **between** `before add` and `after add`. Emitting is synchronous.
2. `queue full` printed only once even though the queue stayed full — that is `once`.
3. `events.once(emitter, name)` turns a single future event into a promise, which is
   the cleanest way to `await` something like a server's `'listening'` event.

## The special `'error'` event

If an emitter emits `'error'` and **no** `'error'` listener is attached, Node throws the
error — which, outside a `try`, crashes the process:

```text
node:events:...
      throw er; // Unhandled 'error' event
      ^
```

This is why you must always attach an `'error'` handler to sockets, streams, and
child processes. An unhandled network hiccup on one connection should not take down
your whole server.

## Consuming events as an async iterator

`events.on(emitter, name)` returns an async iterator of event argument arrays:

```js
import { on } from 'node:events';

const ac = new AbortController();
setTimeout(() => ac.abort(), 5_000);

try {
  for await (const [order] of on(q, 'added', { signal: ac.signal })) {
    await saveToDatabase(order);   // processes events one at a time
  }
} catch (err) {
  if (err.name !== 'AbortError') throw err;
}
```

Events emitted while the loop body is awaiting are buffered and delivered in order.

## EventTarget: the web-standard cousin

Node also implements the browser's `EventTarget`/`Event` (used by `AbortSignal`, for
example). It uses `addEventListener` and `dispatchEvent`, passes a single `Event`
object instead of arbitrary arguments, and does *not* have the special `'error'`
behavior. Use `EventEmitter` for Node-style APIs; you will meet `EventTarget` mostly via
`AbortSignal` and web APIs such as `WebSocket`.

## How It Actually Works

An `EventEmitter` is essentially a dictionary from event name to a listener (or array of
listeners), stored in `this._events`. `on` pushes to the array. `emit(name, ...args)`
looks up the array, **copies** it (so listeners added or removed during emission do not
affect this round), and calls each listener in registration order with `this` set to
the emitter. That's all: no queue, no event loop involvement, no asynchrony.

So when you see asynchronous behavior around events — a socket emitting `'data'` "later"
— it's because *libuv* called into JavaScript later (in the poll phase), and that
callback called `emit`. The emitter itself is just a synchronous fan-out.

Consequences:

- A slow or throwing listener affects the emitter's caller directly. If a listener
  throws, the exception propagates out of `emit()`, and remaining listeners do not run.
- `once` wraps your function in a small wrapper that removes itself and then calls you.
- `events.once()` attaches a `once` listener that resolves a promise, plus an `'error'`
  listener that rejects it.

Each emitter has a **max listeners** threshold (10 by default). Adding an 11th listener
for the same event prints a `MaxListenersExceededWarning`. It is not a hard limit; it's a
leak detector. The common real cause is attaching a listener inside a function that runs
per request (for example, `process.on('exit', ...)` inside a handler) and never removing
it.

If a listener is an `async` function that rejects, the rejection is not an exception
thrown from `emit` — it becomes an unhandled rejection. Construct the emitter with
`{ captureRejections: true }` to route such rejections to the `'error'` event instead.

## Common mistakes

- **No `'error'` listener** on sockets, streams, or custom emitters → process crash.
- **Adding listeners in a loop or per request** without removing them → memory leak and
  `MaxListenersExceededWarning`. Raising the limit with `setMaxListeners` hides the bug.
- **Assuming `emit` is async** and relying on listeners running "after" the current
  code.
- **Emitting in the constructor.** Listeners can't be attached yet, so the event is
  lost. Defer with `process.nextTick(() => this.emit('ready'))` if you must.
- **Heavy work inside listeners** that blocks the emitter's caller.

## Exercise

1. Build a `Timer` class that extends `EventEmitter` and emits `'tick'` every second with
   the elapsed seconds, and `'done'` after N ticks. Use `events.once(timer, 'done')` to
   await completion.
2. Write a listener that throws and register it before a normal listener. Emit the event
   inside `try/catch` and observe which listeners ran.
3. Write a function that adds a listener every time it's called and call it 12 times.
   Read the warning, then fix the leak properly instead of raising the limit.
