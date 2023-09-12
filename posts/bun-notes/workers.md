---
created: 2023-09-12T02:31:24 (UTC -05:00)
tags: []
source: https://bun.sh/docs/api/workers
author: 
---

# Workers – API | Bun Docs

> ## Excerpt
> Run code in a separate thread with Bun's native Worker API.

---
**🚧** — The `Worker` API is still experimental and should not be considered ready for production.

[`Worker`](https://developer.mozilla.org/en-US/docs/Web/API/Worker) lets you start and communicate with a new JavaScript instance running on a separate thread while sharing I/O resources with the main thread.

Bun implements a minimal version of the [Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) with extensions that make it work better for server-side use cases. Like the rest of Bun, `Worker` in Bun support CommonJS, ES Modules, TypeScript, JSX, TSX and more out of the box. No extra build steps are necessary.

## [Creating a `Worker`](https://bun.sh/docs/api/workers#creating-a-worker)

Like in browsers, [`Worker`](https://developer.mozilla.org/en-US/docs/Web/API/Worker) is a global. Use it to create a new worker thread.

From the main thread:

Main thread

```
const workerURL = new URL("worker.ts", import.meta.url).href;
const worker = new Worker(workerURL);

worker.postMessage("hello");
worker.onmessage = event => {
  console.log(event.data);
};
```

Worker thread:

worker.ts (Worker thread)

```
self.onmessage = (event: MessageEvent) => {
  console.log(event.data);
  postMessage("world");
};
```

You can use `import`/`export` syntax in your worker code. Unlike in browsers, there's no need to specify `{type: "module"}` to use ES Modules.

To simplify error handling, the initial script to load is resolved at the time `new Worker(url)` is called.

```
const worker = new Worker("/not-found.js");
// throws an error immediately
```

The specifier passed to `Worker` is resolved relative to the project root (like typing `bun ./path/to/file.js`).

### [`"open"`](https://bun.sh/docs/api/workers#open)

The `"open"` event is emitted when a worker is created and ready to receive messages. This can be used to send an initial message to a worker once it's ready. (This event does not exist in browsers.)

```
const worker = new Worker(new URL("worker.ts", import.meta.url).href);

worker.addEventListener("open", () => {
  console.log("worker is ready");
});
```

Messages are automatically enqueued until the worker is ready, so there is no need to wait for the `"open"` event to send messages.

## [Messages with `postMessage`](https://bun.sh/docs/api/workers#messages-with-postmessage)

To send messages, use [`worker.postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Worker/postMessage) and [`self.postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage). This leverages the [HTML Structured Clone Algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm).

```
// On the worker thread, `postMessage` is automatically "routed" to the parent thread.
postMessage({ hello: "world" });

// On the main thread
worker.postMessage({ hello: "world" });
```

To receive messages, use the [`message` event handler](https://developer.mozilla.org/en-US/docs/Web/API/Worker/message_event) on the worker and main thread.

```
// Worker thread:
self.addEventListener("message", event => {
  console.log(event.data);
});
// or use the setter:
// self.onmessage = fn

// if on the main thread
worker.addEventListener("message", event => {
  console.log(event.data);
});
// or use the setter:
// worker.onmessage = fn
```

## [Terminating a worker](https://bun.sh/docs/api/workers#terminating-a-worker)

A `Worker` instance terminates automatically once it's event loop has no work left to do. Attaching a `"message"` listener on the global or any `MessagePort`s will keep the event loop alive. To forcefully terminate a `Worker`, call `worker.terminate()`.

```
const worker = new Worker(new URL("worker.ts", import.meta.url).href);

// ...some time later
worker.terminate();
```

This will cause the worker's to exit as soon as possible.

### [`process.exit()`](https://bun.sh/docs/api/workers#process-exit)

A worker can terminate itself with `process.exit()`. This does not terminate the main process. Like in Node.js, `process.on('beforeExit', callback)` and `process.on('exit', callback)` are emitted on the worker thread (and not on the main thread), and the exit code is passed to the `"close"` event.

### [`"close"`](https://bun.sh/docs/api/workers#close)

The `"close"` event is emitted when a worker has been terminated. It can take some time for the worker to actually terminate, so this event is emitted when the worker has been marked as terminated. The `CloseEvent` will contain the exit code passed to `process.exit()`, or 0 if closed for other reasons.

```
const worker = new Worker(new URL("worker.ts", import.meta.url).href);

worker.addEventListener("close", event => {
  console.log("worker is being closed");
});
```

This event does not exist in browsers.

## [Managing lifetime](https://bun.sh/docs/api/workers#managing-lifetime)

By default, an active `Worker` will keep the main (spawning) process alive, so async tasks like `setTimeout` and promises will keep the process alive. Attaching `message` listeners will also keep the `Worker` alive.

### [`worker.unref()`](https://bun.sh/docs/api/workers#worker-unref)

To stop a running worker from keeping the process alive, call `worker.unref()`. This decouples the lifetime of the worker to the lifetime of the main process, and is equivlent to what Node.js' `worker_threads` does.

```
const worker = new Worker(new URL("worker.ts", import.meta.url).href);
worker.unref();
```

Note: `worker.unref()` is not available in browers.

### [`worker.ref()`](https://bun.sh/docs/api/workers#worker-ref)

To keep the process alive until the `Worker` terminates, call `worker.ref()`. A ref'd worker is the default behavior, and still needs something going on in the event loop (such as a `"message"` listener) for the worker to continue running.

```
const worker = new Worker(new URL("worker.ts", import.meta.url).href);
worker.unref();
// later...
worker.ref();
```

Alternatively, you can also pass an `options` object to `Worker`:

```
const worker = new Worker(new URL("worker.ts", import.meta.url).href, {
  ref: false,
});
```

Note: `worker.ref()` is not available in browers.

## [Memory usage with `smol`](https://bun.sh/docs/api/workers#memory-usage-with-smol)

JavaScript instances can use a lot of memory. Bun's `Worker` supports a `smol` mode that reduces memory usage, at a cost of performance. To enable `smol` mode, pass `smol: true` to the `options` object in the `Worker` constructor.

```
const worker = new Worker("./i-am-smol.ts", {
  smol: true,
});
```

What does `smol` mode actually do?
