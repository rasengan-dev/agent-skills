---
name: rasengan-futon-routing
description: Router and radix-tree dispatch patterns for @rasenganjs/futon. Covers the Router class (get/post/put/patch/delete/head/options), router.group(prefix, { middlewares }, callback) and nested SubRouter groups, route-level middleware, the RasenganTreeRouter radix-tree dispatch and its path pattern types (:param required, :param? optional, :param* named wildcard, bare * catch-all, match priority), and body parsing helpers (parseJson, parseUrlEncoded, parseFormData, parseText, parseBody). Use when defining routes, route groups, or parsing request bodies in a @rasenganjs/futon app.
license: MIT
metadata:
  author: Dilane Kombou
  framework: futon
  version: "1.0.0-beta.0"
---

# @rasenganjs/futon Routing Patterns

## When to Activate

- Registering routes directly on a `Router` instance (as opposed to `app.get(...)` shortcuts)
- Grouping routes under a shared prefix and/or shared middleware with `.group()`
- Nesting route groups
- Choosing a path pattern: required param, optional param, named wildcard, or catch-all
- Reasoning about which of two overlapping routes matches a given URL
- Parsing a request body manually (outside `bodyParser()`) with `parseJson`/`parseUrlEncoded`/`parseFormData`/`parseText`/`parseBody`

## The Router Class

```ts
import { Router } from '@rasenganjs/futon';
import { json } from '@rasenganjs/futon';

const router = new Router();

router.get('/users', async () => json({ users: [] }));
router.post('/users', async (ctx) => json({ created: true }));
router.get('/users/:id', async (ctx) => json({ id: ctx.params.id }));
router.delete('/users/:id', async (ctx) => json({ deleted: ctx.params.id }));

app.use(router.middleware());
```

Rules:
- 7 HTTP method shortcuts: `get`, `post`, `put`, `patch`, `delete`, `head`, `options` — each is `(pattern: string, handler: (ctx) => Promise<Response>) => this`
- `router.middleware()` produces a single `Middleware` you pass to `app.use()`; `Futon` does this internally for its own built-in router, so you only call it yourself when composing a standalone `Router`
- `router.middleware()` snapshots the currently-registered routes at call time — routes added *after* calling `.middleware()` on that router are not picked up by that snapshot (this does not apply to `app.get/post/...`, which go through `Futon`'s own auto-invalidating cache)
- No matching route + no matching method on any route → falls through to `next()` (typically the app's 404 handler)
- A path matches under a **different** HTTP method than the request → the router short-circuits with `405` and an `Allow` header listing the methods that do match, instead of falling through
- `router.routesCount()` returns the total registered route count (debugging)

## Route-Level Middleware

```ts
router.use(authMiddleware);   // applies to every route registered AFTER this call
router.get('/admin', async (ctx) => json({ ok: true }));  // protected
```

Rules:
- `router.use(...middlewares)` pushes onto a stack that new `add()` calls snapshot — middleware registered before a route applies to it; middleware registered after does not
- Route-level middleware runs in the same onion model as global middleware, nested inside it: `app.use()` globals → route-level `.use()` → handler

## Route Groups

```ts
router.group('/api/v1', { middlewares: [auth] }, (api) => {
  api.get('/users', listUsers);     // → GET /api/v1/users, runs auth first
  api.post('/users', createUser);   // → POST /api/v1/users, runs auth first
});

// Shorthand without options — no group middleware, just a prefix
router.group('/public', (pub) => {
  pub.get('/status', getStatus);    // → GET /public/status
});

// Nested groups — prefixes and middleware compose
router.group('/api', (api) => {
  api.group('/v2', { middlewares: [rateLimiter] }, (v2) => {
    v2.get('/status', getStatusV2); // → GET /api/v2/status, runs rateLimiter
  });
});
```

`Futon.group()` mirrors this on the app instance directly:

```ts
app.group('/api/v1', { middlewares: [auth] }, (api) => {
  api.get('/users', listUsers);
});
```

Rules:
- Two call shapes: `group(prefix, callback)` (no shared middleware) or `group(prefix, { middlewares }, callback)`
- Group middleware only applies to routes registered inside that callback — it does not leak to sibling routes registered outside the group (verified: group middleware stack is restored to its pre-group depth after the callback returns)
- Nested `group()` calls combine prefixes (`/api` + `/v2` → `/api/v2`) and accumulate middleware from every enclosing group
- The callback receives a `Router`-typed object (actually a `SubRouter` internally) — it supports the same `.get/post/.../group/.use` API, so groups nest to any depth

## Path Patterns & Radix-Tree Dispatch

The `Router` dispatches through `RasenganTreeRouter`, a radix (compressed trie) tree — O(k) lookup where k is the number of URL segments, independent of how many routes are registered (not a linear scan).

```ts
router.get('/users/:id', handler);          // required — /users/42 → { id: "42" }
router.get('/users/:id?', handler);         // optional — /users AND /users/42 both match
router.get('/files/:path*', handler);       // named wildcard — /files/a/b/c → { path: "a/b/c" }
router.get('/static/*', handler);           // bare catch-all — /static/app.js → { _: "app.js" }
router.get('/about', handler);              // static
```

Match priority at each path segment (highest to lowest):

1. **Static** — exact segment match
2. **`:param`** — required dynamic segment
3. **`:param?`** — optional dynamic segment (tried skip-first, then consume: for `/users/:id?/posts` matching `/users/posts`, the router first tries treating `posts` as the literal next segment before trying it as the `id` value)
4. **`*` / `:param*`** — wildcard/catch-all, matches all remaining segments greedily

Rules:
- All captured param values are URL-decoded automatically
- Trailing slashes are normalized away before matching (root `/` is preserved)
- A bare `*` with no name stores its capture under the params key `"_"` — access it as `ctx.params._`
- `:param*` is *named* and greedy — it captures every remaining segment joined by `/` under the given name, not just one
- Static routes always win over dynamic ones at the same position, so `/users/me` and `/users/:id` can coexist safely with `/users/me` matching first
- If you need this radix tree directly (rare — usually you use `Router`), it's exported as `RasenganTreeRouter<T>` with `.add(pattern, handler)` / `.match(pathname)` returning `TreeMatchResult<T>` (`{ handler?, params }`)

## Body Parsing Helpers

```ts
import { parseJson, parseUrlEncoded, parseFormData, parseText, parseBody } from '@rasenganjs/futon';

router.post('/upload-json', async (ctx) => {
  const data = await parseJson<{ name: string }>(ctx.request);
  return json({ received: data });
});

router.post('/upload-form', async (ctx) => {
  const form = await parseFormData(ctx.request);      // native FormData
  const file = form.get('avatar');                    // instanceof File for uploads
  return json({ ok: true });
});

router.post('/anything', async (ctx) => {
  const body = await parseBody(ctx.request);           // auto-detects by Content-Type
  return json({ body });
});
```

- `parseJson<T>(request)` — JSON, throws `SyntaxError` on invalid JSON
- `parseUrlEncoded(request)` — `application/x-www-form-urlencoded` → `Record<string, string>`
- `parseFormData(request)` — `multipart/form-data` → native `FormData`
- `parseText(request)` — raw text body
- `parseBody(request)` — inspects `Content-Type` and dispatches to one of the above (falls back to text)

Rules:
- **The Request body is a single-use `ReadableStream` — call exactly one of these per request, and only once.** Calling any parser twice on the same request throws (the stream is already consumed)
- If you use `bodyParser()` middleware globally, do not also call these parsers manually on the same request in a handler — the body is already gone; read `ctx.get('body')` / `ctx.body` instead
- Prefer these standalone functions over manual `await request.json()` when you want consistent error handling and auto-detection via `parseBody`
