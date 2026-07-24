---
name: rasengan-futon-adapters
description: How to run a @rasenganjs/futon app on a real port or edge runtime. Covers the RuntimeAdapter interface (serve, close, watch, assets) from @rasenganjs/runtime, the platform adapters NodeDevAdapter/NodeProdAdapter, BunDevAdapter/BunProdAdapter, and WorkerdProdAdapter (imported from @rasenganjs/runtime/adapters/node|bun|workerd), and futon's own bridge adapters toExpressHandler(app, runtime?) and toWinterCgHandler(app, defaultRuntime?) for Cloudflare Workers/Bun/Deno-style (req, env, ctx) signatures. Use when wiring a Futon instance up to Node, Bun, an Express app, or a WinterCG/Workers-style runtime.
license: MIT
metadata:
  author: Dilane Kombou
  framework: rasengan
  version: "1.0.0-beta.0"
---

# @rasenganjs/futon Adapter Patterns

## When to Activate

- Getting a `Futon` app listening on an actual port
- Choosing between `@rasenganjs/runtime`'s platform adapters (Node/Bun/Workerd) and futon's own `toExpressHandler`/`toWinterCgHandler` bridges
- Mounting a `Futon` app inside an existing Express app
- Deploying a `Futon` app to Cloudflare Workers, Deno, or another WinterCG-style `(req, env, ctx)` runtime
- Wiring dev-mode file watching or graceful shutdown around a `Futon` app

## The Core Pattern

A `Futon` instance is transport-agnostic — it only knows `.fetch(request, runtime) => Promise<Response>`. Something else has to bind that to an actual server. That "something else" is always one of:

1. A **`RuntimeAdapter`** from `@rasenganjs/runtime` (Node, Bun, or Workerd) — the primary, full-featured path (dev server, watch, assets, env loading, lifecycle)
2. A **bridge adapter** from `@rasenganjs/futon` itself — `toExpressHandler` (mount inside an existing Express app) or `toWinterCgHandler` (adapt to a raw `(req, env, ctx)`-style runtime with no `RuntimeAdapter` machinery)

```ts
// app.mjs — build the app, transport-agnostic
import { Futon, json, html, logger, bodyParser } from '@rasenganjs/futon';

const app = new Futon();

app.use(logger());
app.use(bodyParser());

app.get('/', () => html('<h1>Hello</h1>'));
app.get('/hello/:name', (ctx) => json({ message: `Hello, ${ctx.params.name}!` }));
app.onError((err) => json({ error: err.message }, { status: 500 }));
app.notFound(() => html('<h1>404 — not found</h1>'));

export default app;
```

```ts
// server.mjs — bind it to Node
import { NodeDevAdapter } from '@rasenganjs/runtime/adapters/node';
import app from './app.mjs';

const adapter = new NodeDevAdapter({ port: 5330 });
adapter.serve(app);
```

Rules:
- Keep the `Futon` app (routes/middleware) and the adapter wiring (which runtime, which port) in separate modules — this is what makes the same app runnable under Node, Bun, or a serverless edge function unchanged
- `@rasenganjs/futon` has zero runtime dependencies and never imports Node built-ins; `@rasenganjs/runtime` is the package that knows about `node:fs`, `node:http`, Bun globals, etc. — install/import it separately

## The `RuntimeAdapter` Interface

From `@rasenganjs/runtime` (the platform abstraction package — a sibling of `@rasenganjs/futon`, `packages/platform/runtime/`):

```ts
import type { RuntimeAdapter, ServeOptions, Assets } from '@rasenganjs/runtime';

interface RuntimeAdapter<T = any> {
  serve(app?: T | null, options?: ServeOptions): Promise<void>;
  close(): Promise<void>;
  watch?(path: string, callback: () => void): () => void;
  assets: Assets; // get/load/write/delete/list
}
```

Rules:
- `serve(app, options)` — starts the HTTP server bound to the given `Futon` instance; throws if `app` is not provided (all current adapters require in-process mode)
- `close()` — stops the server; Node/Bun dev adapters also call `app.destroy()` (firing `onDestroy` hooks) before tearing down
- `watch` is optional — implemented by `NodeDevAdapter`/`BunDevAdapter`, absent on prod adapters and on `WorkerdProdAdapter` (no filesystem watching on the edge)
- `assets` is always present: `NodeAssets`/`BunAssets` read/write the local filesystem in dev, are read-only in prod adapters, and are all no-ops on `WorkerdProdAdapter` (no local filesystem in Workers — use KV/R2 instead)
- The main `@rasenganjs/runtime` entry only exports **types** plus `parseEnv`/`getEnvFileNames`/`detectRuntime` — concrete adapter classes live behind separate sub-path exports to avoid a hard dependency from `@rasenganjs/runtime` back onto `@rasenganjs/futon`

## Node — `@rasenganjs/runtime/adapters/node`

```ts
import { NodeDevAdapter } from '@rasenganjs/runtime/adapters/node';
import app from './app.mjs';

const adapter = new NodeDevAdapter({ port: 5330, host: '0.0.0.0', rootDir: process.cwd() });
await adapter.serve(app);
```

```ts
// Production
import { NodeProdAdapter } from '@rasenganjs/runtime/adapters/node';

const adapter = new NodeProdAdapter({ port: 8080, rootDir: './dist' });
await adapter.serve(app);
```

Rules:
- `NodeDevAdapterOptions` / `NodeProdAdapterOptions`: `{ port?, host?, rootDir? }` — dev defaults to port `5200`, host `0.0.0.0`
- `serve()` calls `app.configureServer({ preset: 'node', mode, port, host, rootDir })` and `app.loadEnv(...)` for you before starting — this is how `ctx.runtime.server` and env vars get populated inside handlers
- `NodeDevAdapter` also supports `options.autoRestart` (nodemon-style child-process spawning) — when set, `serve()`'s returned promise stays pending until `close()` is called
- Other named exports from this sub-path: `NodeAssets`, `NodeWatcher`, `startNodeServer`, `loadNodeEnvFiles`, `createNodeUpgradeHandler` (WebSocket upgrades) — reach for `NodeDevAdapter`/`NodeProdAdapter` first; the rest are building blocks it uses internally

## Bun — `@rasenganjs/runtime/adapters/bun`

```ts
import { BunDevAdapter } from '@rasenganjs/runtime/adapters/bun';
import app from './app.mjs';

const adapter = new BunDevAdapter({ port: 3000 });
await adapter.serve(app);
```

Rules:
- Mirrors the Node adapter shape exactly: `BunDevAdapter`/`BunProdAdapter`, same `{ port?, host?, rootDir? }` options, same `serve()`/`close()`/`watch()` contract
- Runs the HTTP server via `Bun.serve()` internally (`startBunServer`) and file assets via `Bun.file`/`Bun.write`

## Workerd (Cloudflare Workers) — `@rasenganjs/runtime/adapters/workerd`

```ts
// Service-worker format (default)
import { WorkerdProdAdapter } from '@rasenganjs/runtime/adapters/workerd';
import app from './app.mjs';

const adapter = new WorkerdProdAdapter();
adapter.serve(app); // registers self.addEventListener('fetch', ...)
```

```ts
// ES modules format
import { WorkerdProdAdapter } from '@rasenganjs/runtime/adapters/workerd';
import app from './app';

const adapter = new WorkerdProdAdapter({ passthrough: true });
await adapter.serve(app);
export default { fetch: adapter.fetchHandler };
```

Rules:
- Production-only — there is no `WorkerdDevAdapter`; local development uses `wrangler dev`/miniflare, not this package
- `watch()` is not implemented (no `?` on the interface here — it's simply absent) and `assets` is always a no-op stub — serve static files via Workers KV/R2 instead
- `{ passthrough: true }` skips registering the `fetch` event listener and instead exposes `adapter.fetchHandler` for you to export yourself — required for the ES-modules Worker format (`export default { fetch }`)
- Without `passthrough`, `serve()`'s returned promise never resolves (workerd keeps the event loop alive via the listener) — do not `await` it expecting completion

## Bridge: `toExpressHandler(app, runtime?)`

Use when you already have an Express app and want to mount `Futon` as one route/middleware inside it, rather than letting a `RuntimeAdapter` own the whole HTTP server.

```ts
import express from 'express';
import { Futon, toExpressHandler } from '@rasenganjs/futon';

const app = new Futon();
app.get('/api/health', async () => new Response('ok'));

const expressApp = express();
expressApp.use(toExpressHandler(app));
expressApp.listen(3000);
```

Rules:
- Returns an Express `(req, res, next)` handler — builds a Web API `Request` from the Express request, calls `app.fetch(request, runtime)`, then streams the `Response` back onto the Express response
- Errors thrown while building the request or during `.fetch()` are passed to Express's `next(error)`, not swallowed
- Second argument `runtime?: RuntimeContext` lets you inject extra `env` merged into every request's `ctx.runtime`
- This bridge does not call `app.configureServer()`/`app.loadEnv()`/`app.init()` for you — those are `RuntimeAdapter` responsibilities; if you need them while embedding in Express, call them yourself before mounting

## Bridge: `toWinterCgHandler(app, defaultRuntime?)`

Use for raw WinterCG-style runtimes (Cloudflare Workers, Bun, Deno, service workers) when you want a plain `fetch`-shaped function without going through a `RuntimeAdapter` at all.

```ts
// Cloudflare Workers (service worker style import)
import { Futon, toWinterCgHandler } from '@rasenganjs/futon';

const app = new Futon();
app.get('/hello', async () => new Response('Hello!'));

export default { fetch: toWinterCgHandler(app) };
```

```ts
// Bun / Deno
Bun.serve({ fetch: toWinterCgHandler(app) });
```

Rules:
- Returns `(request: Request, envOrCtx?, platformCtx?) => Promise<Response>` and normalizes three calling conventions automatically: Cloudflare Workers `(request, env, ctx)`, Deno-style `(request, { env, ctx })`, and plain `(request, env)`
- All `env` values are coerced to strings and merged into `ctx.runtime.env` (with `defaultRuntime.env` merged first, so per-call env wins on conflicts)
- If the platform provides `ctx.waitUntil` and the response carries an `X-Rasengan-Background` header, the handler calls `waitUntil` for you — otherwise it's a no-op
- Prefer `WorkerdProdAdapter` over this bridge when you also want `app.configureServer()`/`app.init()`/lifecycle wiring; reach for `toWinterCgHandler` when you want the bare fetch function with nothing else attached
