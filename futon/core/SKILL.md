---
name: rasengan-futon-core
description: Core API patterns for @rasenganjs/futon, the zero-dependency WinterCG-compatible HTTP app class. Covers the Futon class (new Futon(), .use(), HTTP method shortcuts, .fetch()), the Middleware onion model via compose(), the five built-in middleware (bodyParser, cors, logger, requestId, compress), Context (ctx.params, ctx.query, ctx.state, ctx.set/get, ctx.res, ctx.runtime), response helpers (json, text, html, redirect, status, notFound, streamResponse, nodeStreamToResponse), cookies (setCookie, clearCookie, parseCookies, getCookie), the HookSystem (beforeRequest/afterResponse/onError), and the HttpError hierarchy. Use when writing or reviewing code that imports from '@rasenganjs/futon'.
license: MIT
metadata:
  author: Dilane Kombou
  framework: futon
  version: "1.0.0-beta.0"
---

# @rasenganjs/futon Core Patterns

## When to Activate

- Creating a `Futon` app instance or registering routes/middleware
- Writing custom middleware and reasoning about execution order
- Using built-in middleware: `bodyParser`, `cors`, `logger`, `requestId`, `compress`
- Reading/writing request-scoped state via `Context`
- Building responses with `json`, `text`, `html`, `redirect`, or `ctx.res`
- Setting or reading cookies
- Registering lifecycle hooks (`beforeRequest`, `afterResponse`, `onError`)
- Throwing or handling `HttpError` and its subclasses
- Any code importing from `@rasenganjs/futon`

## The Futon App Class

```ts
import { Futon, json, logger, cors } from '@rasenganjs/futon';

const app = new Futon();

app.use(logger());
app.use(cors());

app.get('/api/health', async (ctx) => json({ status: 'ok' }));

app.onError(async (error, ctx) => {
  console.error(error);
  return json({ error: error.message }, { status: 500 });
});

app.notFound(async (ctx) => json({ error: 'Not Found' }, { status: 404 }));

// Handed to a runtime adapter's .serve() — see rasengan-futon-adapters
export default app;
```

Rules:
- `new Futon()` takes no arguments — it owns an internal `Router`
- Route handlers have the signature `(ctx: Context) => Promise<Response>`
- `app.get/post/put/patch/delete/head/options(pattern, handler)` — all 7 delegate to the internal `Router`
- `app.fetch(request, runtime?)` is the WinterCG entry point — `(Request, RuntimeContext) => Promise<Response>`; adapters call this, you rarely call it directly
- `app.getRouter()` exposes the internal `Router` for programmatic route building
- `app.group(prefix, { middlewares? }, callback)` is a shortcut that delegates to `router.group()` — see `rasengan-futon-routing` for group semantics
- `app.onInit(handler)` / `app.onDestroy(handler)` register app-lifecycle setup/teardown (DB connections, cache warming) — run by the runtime adapter's `serve()`/`close()`, not by `.fetch()`

## Scoped Middleware with `app.use(path, mw)`

```ts
app.use(cors());                      // global — every request
app.use('/api', requireAuth());       // only requests whose path starts with "/api"
```

Rules:
- Passing a string as the first argument to `.use()` scopes that middleware to paths starting with that prefix; everything else falls through via `next()`
- Order matters — middleware registered earlier wraps middleware registered later (onion model, see below)

## Middleware & the Onion Model (`compose()`)

```ts
import type { Middleware } from '@rasenganjs/futon';

const timing: Middleware = async (ctx, next) => {
  const start = Date.now();
  const response = await next();       // downstream (inward)
  const ms = Date.now() - start;       // runs on the way back out
  console.log(`${ctx.request.method} ${ctx.request.url} — ${ms}ms`);
  return response;
};

app.use(timing);
```

```ts
import { compose } from '@rasenganjs/futon';

const chain = compose([mwA, mwB, mwC]); // → single Middleware
```

Rules:
- Signature: `(ctx: Context, next: () => Promise<Response>) => Promise<Response>` — a middleware MUST return the `Response` that eventually bubbles back up
- Code before `next()` runs on the way in; code after `next()` runs on the way out (Koa-style onion)
- Calling `next()` twice in the same middleware throws `Error('next() called multiple times')` — this is enforced, not just discouraged
- `compose([])` (empty array) is a pass-through to whatever `next` fallback is provided
- `Futon` composes `[...globalMiddlewares, router.middleware()]` internally and caches the chain, rebuilding it only when `.use()` or route registration changes the app — you never call `compose()` yourself unless building custom pipelines

## Built-in Middleware

### `bodyParser(options?)`

```ts
import { bodyParser } from '@rasenganjs/futon';

app.use(bodyParser());               // stores parsed body on ctx.state.body / ctx.body

app.post('/api/data', async (ctx) => {
  const body = ctx.get('body');      // same key as options.key (default "body")
  return json({ received: body });
});
```

```ts
interface BodyParserOptions {
  key?: string;              // ctx.state key, default "body"
  maxSize?: number;          // bytes; enables streaming byte-count enforcement (413 on overflow)
  allowedTypes?: string[];   // Content-Type allowlist substrings; non-matching bodies are left unparsed
  skipMultipart?: boolean;   // leave multipart/form-data unread for a downstream fileUpload() middleware
}
```

Rules:
- Skips GET, HEAD, and DELETE requests automatically
- Parses eagerly (before `next()`) — this is the only safe pattern because the body `ReadableStream` can only be read once
- On a parse error, `ctx.state[key]` is set to `undefined` instead of throwing — check for `undefined` downstream
- Also sets `ctx.body` to the same parsed value

### `cors(options?)`

```ts
import { cors } from '@rasenganjs/futon';

app.use(cors({ origin: 'https://myapp.com', credentials: true, maxAge: 86400 }));
```

```ts
interface CORSOptions {
  origin?: string | string[];      // default "*"
  methods?: string;                // default "GET, POST, PUT, PATCH, DELETE, OPTIONS"
  allowedHeaders?: string;         // default "Content-Type, Authorization"
  exposedHeaders?: string;
  credentials?: boolean;
  maxAge?: number;                 // seconds, preflight cache
  optionsStatus?: number;          // default 204
}
```

Rules:
- OPTIONS preflight requests short-circuit — the middleware chain never runs for them
- Existing `Access-Control-Allow-Origin`/`-Credentials`/`-Expose-Headers` headers set by an inner handler are never overwritten

### `logger(options?)`

```ts
import { logger } from '@rasenganjs/futon';

app.use(logger());                                   // colored single-line log
app.use(logger({ skip: ['/health'], methodPadding: 8 }));
app.use(logger({ log: (entry) => pino.info(entry) })); // structured LogEntry
```

```ts
interface LoggerOptions {
  log?: (entry: LogEntry) => void;   // custom sink; entry: { method, pathname, search, status, duration, size }
  skip?: string[];                   // path-prefix denylist
  methodPadding?: number;            // default 6
}
```

Rules:
- Logs even when downstream throws — `status` is `0` on the error path, and the error is re-thrown after logging
- `size` comes from the response's `Content-Length` header; `null` if absent

### `requestId(options?)`

```ts
import { requestId } from '@rasenganjs/futon';

app.use(requestId());
app.use(async (ctx, next) => {
  console.log('handling', ctx.get('requestId'));
  return next();
});
```

```ts
interface RequestIdOptions {
  header?: string;          // default "X-Request-Id"
  stateKey?: string;        // default "requestId"
  generator?: () => string; // default crypto.randomUUID(), falls back to a timestamp-based id
}
```

Rules:
- Reuses an incoming `X-Request-Id` header if present (distributed tracing); otherwise generates one
- Never overwrites an `X-Request-Id` already set on the outgoing response

### `compress(options?)`

```ts
import { compress } from '@rasenganjs/futon';

app.use(compress());                                  // gzip/brotli/deflate over 1KB
app.use(compress({ threshold: 512, encodings: ['gzip'] }));
```

```ts
interface CompressOptions {
  threshold?: number;    // bytes, default 1024
  encodings?: string[];  // priority order, default ["br", "gzip", "deflate"]
}
```

Rules:
- Uses the Web API `CompressionStream` — if unavailable in the runtime, the middleware silently no-ops and returns the response uncompressed
- Skips responses that already carry a `Content-Encoding` header, have a `null` body, or are below `threshold`
- Only compresses an encoding the client actually advertises via `Accept-Encoding`

Other built-ins exist beyond this skill's scope — `basicAuth`, `bearerToken`, `bodyLimit`, and `fileUpload` (upload middleware, with `DiskStorage` importable from the `@rasenganjs/futon/upload/disk` subpath to keep `node:fs` out of WinterCG bundles). Check `packages/framework/futon/src/middlewares/` for their option shapes before using them.

## Context

```ts
app.post('/users/:id', async (ctx) => {
  ctx.params.id;                 // path params from the Router
  ctx.query.page;                // ctx.query('page') also works (callable + indexable)
  ctx.set('user', { id: 1 });    // write to the state bag
  const user = ctx.get('user');  // read it back (typed via generic)
  ctx.runtime.env;                // Record<string, string> — platform env vars
  ctx.runtime.server?.port;       // ServerInfo set by the runtime adapter
  return ctx.res.status(201).json({ user });
});
```

Rules:
- `ctx.request` and `ctx.req` are the same Web API `Request` — never mutate it; use `ctx.state` to pass data between middleware and handlers
- `ctx.query` is lazily parsed on first access and cached — safe to leave unread with no parsing cost
- `ctx.set(key, value)` / `ctx.get<T>(key)` are the sanctioned channel for middleware → middleware / middleware → handler communication (e.g. auth middleware sets `ctx.set('user', ...)`, the handler reads `ctx.get('user')`)
- `ctx.res` and `ctx.response` are the same lazily-created `ResponseBuilder` — chainable: `ctx.res.status(404).send('Not found')`, `ctx.res.redirect('/login')`, `ctx.res.json(data)`. Once you call a terminal method (`json`/`send`/`html`/`redirect`/`stream`), the builder freezes further `.status()`/`.header()` calls
- `ctx.body` holds whatever `bodyParser()` (or a validator) stored — `undefined` until that middleware runs

## Response Helpers

```ts
import { json, text, html, redirect, status, notFound, streamResponse, nodeStreamToResponse } from '@rasenganjs/futon';

json({ ok: true }, { status: 201 });     // application/json Response
text('pong');                            // plain text Response
html('<h1>Hi</h1>');                     // text/html; charset=utf-8 Response
redirect('/login');                      // 302 by default
redirect('/login', 301);                 // explicit status
status(204);                             // status-only, no body
notFound();                              // shorthand for status(404, 'Not Found')

// SSR streaming (e.g. React renderToPipeableStream)
streamResponse(webReadableStream, { status: 200 });
nodeStreamToResponse(nodePipeableStream, { status: 200 }); // bridges a Node stream
```

Rules:
- All helpers return a plain `Response` — nothing framework-specific leaks into the return type
- Prefer these over `new Response(...)` directly for consistent headers and (for `json`/`text`/`html`) a raw-body fast-path tag adapters can use
- Use `ctx.res.*` inside a handler when you want a fluent, stateful build-up (status + headers + body in one chain); use the free functions (`json`, `text`, ...) for one-shot returns

## Cookies

```ts
import { setCookie, clearCookie, parseCookies, getCookie } from '@rasenganjs/futon';

app.get('/login', async (ctx) => {
  const res = json({ ok: true });
  return setCookie(res, 'session', token, { httpOnly: true, secure: true, maxAge: 86400 });
});

app.get('/logout', async (ctx) => clearCookie(json({ ok: true }), 'session'));

app.get('/me', async (ctx) => {
  const session = getCookie(ctx.request, 'session');   // or: parseCookies(ctx.request)['session']
  return json({ session });
});
```

```ts
interface CookieOptions {
  domain?: string;
  path?: string;               // default "/"
  maxAge?: number;              // seconds
  httpOnly?: boolean;
  secure?: boolean;
  sameSite?: 'Strict' | 'Lax' | 'None';
  expires?: Date;
}
```

Rules:
- `setCookie`/`clearCookie` return a **new** `Response` (body/status preserved, `Set-Cookie` appended) — always use the returned value, the original `Response` is not mutated
- `clearCookie` is `setCookie(res, name, '', { ...options, maxAge: 0 })` under the hood
- `parseCookies(request)` / `getCookie(request, name)` read from `ctx.request` — pass `ctx.request`, not `ctx`

## Hook System

```ts
app.hooks.on('beforeRequest', (ctx) => {
  console.log('incoming', ctx.request.method, ctx.request.url);
});

app.hooks.on('afterResponse', (ctx, response) => {
  metrics.record(ctx.request.method, response.status);
});

app.hooks.on('onError', (error, ctx) => {
  Sentry.captureException(error);
});
```

Rules:
- Three hook names only: `beforeRequest(ctx)`, `afterResponse(ctx, response)`, `onError(error, ctx)`
- `HookSystem` also exposes `off(name, handler)`, `emit(name, ...args)`, and `clear()` — `Futon` owns one instance at `app.hooks`
- Hook handler errors are swallowed (caught and dropped) so a broken metrics hook never crashes the request — do your own error handling inside the hook if you need visibility into hook failures
- Handlers run in registration order; async handlers are awaited via `Promise.allSettled`
- `onError` fires **before** the app's `.onError()` handler runs, so monitoring hooks observe every error regardless of how (or whether) it's user-handled

## HttpError Hierarchy

```ts
import { HttpError, NotFoundError, MethodNotAllowedError, InternalServerError } from '@rasenganjs/futon';

app.get('/users/:id', async (ctx) => {
  const user = await db.users.find(ctx.params.id);
  if (!user) throw new NotFoundError(`User ${ctx.params.id} not found`);
  return json(user);
});

app.onError(async (error, ctx) => {
  if (error instanceof HttpError) {
    return json({ error: error.message }, { status: error.status });
  }
  return json({ error: 'Internal Server Error' }, { status: 500 });
});
```

Rules:
- `HttpError(status, message?)` is the base class — every instance carries a numeric `.status`; omitting `message` falls back to a built-in status-text table (e.g. 404 → "Not Found")
- Built-in subclasses: `NotFoundError` (404), `MethodNotAllowedError` (405), `InternalServerError` (500) — each has a sensible default message
- Futon does **not** auto-convert thrown `HttpError`s to a status-coded response — without a custom `app.onError()`, every thrown error (including `HttpError`) produces a generic `text(message, { status: 500 })`. Always register `onError` and branch on `instanceof HttpError` if you want the error's own `.status` honored
