---
created: 2023-09-12T02:32:30 (UTC -05:00)
tags: []
source: https://bun.sh/docs/api/http
author: 
---

# HTTP server – API | Bun Docs

> ## Excerpt
> Bun implements Web-standard fetch, plus a Bun-native API for building fast HTTP servers.

---
The page primarily documents the Bun-native `Bun.serve` API. Bun also implements [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) and the Node.js [`http`](https://nodejs.org/api/http.html) and [`https`](https://nodejs.org/api/https.html) modules.

These modules have been re-implemented to use Bun's fast internal HTTP infrastructure. Feel free to use these modules directly; frameworks like [Express](https://expressjs.com/) that depend on these modules should work out of the box. For granular compatibility information, see [Runtime > Node.js APIs](https://bun.sh/docs/runtime/nodejs-apis).

To start a high-performance HTTP server with a clean API, the recommended approach is [`Bun.serve`](https://bun.sh/docs/api/http#start-a-server-bun-serve).

## [`Bun.serve()`](https://bun.sh/docs/api/http#bun-serve)

Start an HTTP server in Bun with `Bun.serve`.

```
Bun.serve({
  fetch(req) {
    return new Response("Bun!");
  },
});
```

The `fetch` handler handles incoming requests. It receives a [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) object and returns a [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) or `Promise<Response>`.

```
Bun.serve({
  fetch(req) {
    const url = new URL(req.url);
    if (url.pathname === "/") return new Response("Home page!");
    if (url.pathname === "/blog") return new Response("Blog!");
    return new Response("404!");
  },
});
```

To configure which port and hostname the server will listen on:

```
Bun.serve({
  port: 8080, // defaults to $BUN_PORT, $PORT, $NODE_PORT otherwise 3000
  hostname: "mydomain.com", // defaults to "0.0.0.0"
  fetch(req) {
    return new Response("404!");
  },
});
```

To listen on a [unix domain socket](https://en.wikipedia.org/wiki/Unix_domain_socket):

```
Bun.serve({
  unix: "/tmp/my-socket.sock", // path to socket
  fetch(req) {
    return new Response(`404!`);
  },
});
```

## [Error handling](https://bun.sh/docs/api/http#error-handling)

To activate development mode, set `development: true`. By default, development mode is _enabled_ unless `NODE_ENV` is `production`.

```
Bun.serve({
  development: true,
  fetch(req) {
    throw new Error("woops!");
  },
});
```

In development mode, Bun will surface errors in-browser with a built-in error page.

[![](https://bun.sh/images/exception_page.png)](https://bun.sh/images/exception_page.png)

Bun's built-in 500 page

To handle server-side errors, implement an `error` handler. This function should return a `Response` to served to the client when an error occurs. This response will supercede Bun's default error page in `development` mode.

```
Bun.serve({
  fetch(req) {
    throw new Error("woops!");
  },
  error(error) {
    return new Response(`<pre>${error}\n${error.stack}</pre>`, {
      headers: {
        "Content-Type": "text/html",
      },
    });
  },
});
```

The call to `Bun.serve` returns a `Server` object. To stop the server, call the `.stop()` method.

```
const server = Bun.serve({
  fetch() {
    return new Response("Bun!");
  },
});

server.stop();
```

## [TLS](https://bun.sh/docs/api/http#tls)

Bun supports TLS out of the box, powered by [BoringSSL](https://boringssl.googlesource.com/boringssl). Enable TLS by passing in a value for `key` and `cert`; both are required to enable TLS.

```
Bun.serve({
  fetch(req) {
    return new Response("Hello!!!");
  },

  tls: {
    key: Bun.file("./key.pem"),
    cert: Bun.file("./cert.pem"),
  }
});
```

The `key` and `cert` fields expect the _contents_ of your TLS key and certificate, _not a path to it_. This can be a string, `BunFile`, `TypedArray`, or `Buffer`.

```
Bun.serve({
  fetch() {},

  tls: {
    // BunFile
    key: Bun.file("./key.pem"),
    // Buffer
    key: fs.readFileSync("./key.pem"),
    // string
    key: fs.readFileSync("./key.pem", "utf8"),
    // array of above
    key: [Bun.file("./key1.pem"), Bun.file("./key2.pem")],
  },
});
```

If your private key is encrypted with a passphrase, provide a value for `passphrase` to decrypt it.

```
Bun.serve({
  fetch(req) {
    return new Response("Hello!!!");
  },

  tls: {
    key: Bun.file("./key.pem"),
    cert: Bun.file("./cert.pem"),
    passphrase: "my-secret-passphrase",
  }
});
```

Optionally, you can override the trusted CA certificates by passing a value for `ca`. By default, the server will trust the list of well-known CAs curated by Mozilla. When `ca` is specified, the Mozilla list is overwritten.

```
Bun.serve({
  fetch(req) {
    return new Response("Hello!!!");
  },
  tls: {
    key: Bun.file("./key.pem"), // path to TLS key
    cert: Bun.file("./cert.pem"), // path to TLS cert
    ca: Bun.file("./ca.pem"), // path to root CA certificate
  }
});
```

To override Diffie-Helman parameters:

```
Bun.serve({
  // ...
  tls: {
    // other config
    dhParamsFile: "/path/to/dhparams.pem", // path to Diffie Helman parameters
  },
});
```

## [Object syntax](https://bun.sh/docs/api/http#object-syntax)

Thus far, the examples on this page have used the explicit `Bun.serve` API. Bun also supports an alternate syntax.

server.ts

```
import {type Serve} from "bun";

export default {
  fetch(req) {
    return new Response("Bun!");
  },
} satisfies Serve;
```

Instead of passing the server options into `Bun.serve`, `export default` it. This file can be executed as-is; when Bun sees a file with a `default` export containing a `fetch` handler, it passes it into `Bun.serve` under the hood.

## [Streaming files](https://bun.sh/docs/api/http#streaming-files)

To stream a file, return a `Response` object with a `BunFile` object as the body.

```
import { serve, file } from "bun";

serve({
  fetch(req) {
    return new Response(Bun.file("./hello.txt"));
  },
});
```

⚡️ **Speed** — Bun automatically uses the [`sendfile(2)`](https://man7.org/linux/man-pages/man2/sendfile.2.html) system call when possible, enabling zero-copy file transfers in the kernel—the fastest way to send files.

You can send part of a file using the [`slice(start, end)`](https://developer.mozilla.org/en-US/docs/Web/API/Blob/slice) method on the `Bun.file` object. This automatically sets the `Content-Range` and `Content-Length` headers on the `Response` object.

```
Bun.serve({
  fetch(req) {
    // parse `Range` header
    const [start = 0, end = Infinity] = req.headers
      .get("Range") // Range: bytes=0-100
      .split("=") // ["Range: bytes", "0-100"]
      .at(-1) // "0-100"
      .split("-") // ["0", "100"]
      .map(Number); // [0, 100]

    // return a slice of the file
    const bigFile = Bun.file("./big-video.mp4");
    return new Response(bigFile.slice(start, end));
  },
});
```

## [Benchmarks](https://bun.sh/docs/api/http#benchmarks)

Below are Bun and Node.js implementations of a simple HTTP server that responds `Bun!` to each incoming `Request`.

Bun

```
Bun.serve({
  fetch(req: Request) {
    return new Response("Bun!");
  },
  port: 3000,
});
```

Node

```
require("http")
  .createServer((req, res) => res.end("Bun!"))
  .listen(8080);
```

The `Bun.serve` server can handle roughly 2.5x more requests per second than Node.js on Linux.

| Runtime | Requests per second |
| --- | --- |
| Node 16 | ~64,000 |
| Bun | ~160,000 |

[![image](https://user-images.githubusercontent.com/709451/162389032-fc302444-9d03-46be-ba87-c12bd8ce89a0.png)](https://user-images.githubusercontent.com/709451/162389032-fc302444-9d03-46be-ba87-c12bd8ce89a0.png)

## [Reference](https://bun.sh/docs/api/http#reference)

See TypeScript definitions
