---
name: rasengan-server-core
description: Core application patterns for @rasenganjs/server, the modular controller-based backend framework built on @rasenganjs/futon. Covers bootstrap(), ServerApp (.use, .enableCors, .onError, .notFound, .onInit/.onDestroy, .registerModule, .registerPlugin), defineModule({ prefix, middlewares, imports, controllers, providers, exports, global }), the Controller abstract class and routes(router) pattern, Router HTTP verb overloads, dependency injection via Provider/Container (class-identity resolution, { provide, useClass, useValue, deps }, string-name auto-wiring), RasenganServerConfig/defineConfig, and the rasengan-server CLI (dev/build/start). Use when creating or modifying rasengan.server.js, src/main.ts, *.module.ts, or *.controller.ts files in a Rasengan server app.
license: MIT
metadata:
  author: Dilane Kombou
  framework: rasengan server
  version: "1.0.0-beta.0"
---

# @rasenganjs/server Core Patterns

## When to Activate

- Creating or editing `rasengan.server.{js,ts}` or `src/main.ts` (server entry)
- Writing `bootstrap()` callbacks that register modules, middleware, or lifecycle hooks
- Defining a module with `defineModule({ prefix, controllers, providers, imports, ... })`
- Writing a `Controller` subclass with a `routes(router)` method
- Wiring dependency injection between controllers and providers/services
- Configuring the server (port, host, preset, build output) via `defineConfig`
- Running the CLI (`rasengan-server dev|build|start`)

## Bootstrap

`bootstrap()` is the entry point. It creates a `ServerApp`, loads config, runs your callback to register modules/middleware, compiles, and starts the runtime adapter (Node/Bun/workerd):

```ts
// src/main.ts
import { bootstrap } from '@rasenganjs/server';
import appModule from './app.module';

bootstrap(async (app) => {
  app.registerModule(appModule);

  app.notFound(async (ctx) => {
    return ctx.response.status(404).json({ message: 'Not Found' });
  });

  app.onInit(() => {
    console.log('App initialized');
  });

  app.onDestroy(() => {
    console.log('App destroyed');
  });
});
```

Rules:
- `bootstrap()` returns a `ServerHandle` (`{ close(), app }`) — useful for tests or programmatic shutdown
- `bootstrap()` sets up `SIGTERM`/`SIGINT` handlers automatically for graceful shutdown
- Register `ModulePlugin`s (e.g. `@rasenganjs/ws`'s `createWsPlugin()`, `@rasenganjs/queue`'s `createQueuePlugin()`) with `app.registerPlugin()` **before** `app.registerModule()` if that module declares the plugin's extension key (`gateways`, `queues`)

## ServerApp API

| Method | Purpose |
|--------|---------|
| `registerModule(mod \| () => mod)` | Register a `ModuleConfig`, directly or via a factory |
| `registerPlugin(plugin)` | Claim a `defineModule()` extension key (e.g. `'gateways'`) for an ecosystem package |
| `use(middleware)` | Register global middleware (runs before every request) |
| `enableCors(options?)` | Enable CORS; no-arg form uses defaults |
| `onError(handler)` | Custom handler for uncaught exceptions — `(error, ctx) => Promise<Response>` |
| `notFound(handler)` | Custom 404 handler — `(ctx) => Promise<Response>` |
| `onInit(handler)` | Runs once before the server starts accepting requests |
| `onDestroy(handler)` | Runs during graceful shutdown |
| `websocket(path, handlers)` | Register a raw WebSocket route (see the websockets skill) |
| `configureValidation(config)` | Set the schema adapter / error handler (see the validation skill) |

```ts
app.enableCors({ origin: 'https://example.com' });

app.onError(async (error, ctx) => {
  return new Response(JSON.stringify({ error: error.message }), {
    status: 500,
    headers: { 'content-type': 'application/json' },
  });
});
```

Rules:
- Global middleware runs first, then module-level, then controller-level, then route-level, then validation, then the handler
- A JSON/urlencoded/multipart body parser is **always active by default** (registered internally, populates `ctx.body`) — as of `1.0.0-beta.0` there is no working `configureBodyParser()` (it exists only as a commented-out stub in source); don't call it
- `app.compile()` and `app.close()` exist but are invoked internally by `bootstrap()` — call them directly only in tests that bypass `bootstrap()`

## Defining Modules

`defineModule()` groups controllers, providers, and middleware under an optional URL prefix. Modules can import other modules to build a tree:

```ts
// app.module.ts
import { defineModule } from '@rasenganjs/server';
import UserModule from './user.module';
import { PingController } from './ping.controller';

export default defineModule({
  imports: [UserModule],
  controllers: [PingController],
});

// user.module.ts
import { defineModule } from '@rasenganjs/server';
import { UserController } from './user.controller';
import { UserService } from './user.service';

export default defineModule({
  prefix: '/users',
  controllers: [UserController],
  providers: [
    {
      provide: UserController,
      deps: [UserService, 'CONFIG'],
    },
    {
      provide: 'CONFIG',
      useValue: { port: 3000 },
    },
  ],
});
```

`ModuleConfig` fields:

| Field | Type | Description |
|-------|------|--------------|
| `name?` | `string` | Diagnostic name shown in DI error messages |
| `prefix?` | `string` | URL prefix applied to every route the module's controllers register |
| `middlewares?` | `Middleware[]` | Module-scoped middleware |
| `imports?` | `ModuleConfig[]` | Sub-modules to flatten into the tree |
| `controllers?` | `Controller` classes | Controllers to register |
| `providers?` | `(class \| ProviderDefinition)[]` | DI registrations, private to this module unless exported |
| `exports?` | `any[]` | Tokens from `providers` that importing modules may resolve |
| `global?` | `boolean` | Make this module's `exports` visible to every module without an explicit import |

Rules:
- Only tokens listed in `providers` may appear in `exports` — exporting something the module doesn't own throws at compile time
- Imports are flattened depth-first and deduped by object identity; diamond imports are safe
- Ecosystem packages add their own keys (e.g. `gateways`, `queues`) — these only work if the matching `ModulePlugin` was registered via `app.registerPlugin()` first, otherwise `compile()` throws

## Controllers

`Controller` is an abstract class — subclass it and implement `routes(router)`:

```ts
import { Controller, type RouteHandler, type Router } from '@rasenganjs/server';
import { UserService } from './user.service';

export class UserController extends Controller {
  constructor(private userService: UserService) {
    super();
  }

  routes(router: Router) {
    router.get('/', this.findAll);
    router.get('/:id', this.findOne);
  }

  findAll: RouteHandler = async (ctx) => {
    const list = await this.userService.findAll();
    return ctx.res.json(list);
  };

  findOne: RouteHandler = async (ctx) => {
    const user = await this.userService.findById(ctx.params.id);
    if (!user) return ctx.res.status(404).json({ error: 'User not found' });
    return ctx.res.json(user);
  };
}
```

`Router` methods (`get`/`post`/`put`/`patch`/`delete`) accept these argument shapes:

```ts
router.get('/public', handler);
router.get('/admin', authMiddleware, handler);
router.get('/admin', [authMiddleware, rateLimitMiddleware], handler);
// with a schema (see the validation skill), as the LAST argument:
router.post('/users', handler, { body: CreateUserSchema });
router.put('/users/:id', [authMiddleware], handler, { body: UpdateSchema });
```

Rules:
- Every controller must implement `routes(router: Router): void` — `compile()` throws if it's missing
- Declare handlers as class-field arrow functions (`findAll: RouteHandler = async (ctx) => {...}`) so `this` stays bound
- `Controller.middlewares: Middleware[]` (set as a class field) applies to every route in that controller, after module-level middleware and before route-level middleware
- Context inside a handler exposes `ctx.request`/`ctx.req`, `ctx.body`, `ctx.params`, `ctx.query`, `ctx.state`, `ctx.get()/set()`, and a chainable response builder on both `ctx.res` and `ctx.response` (`ctx.res.status(201).json(...)`)
- A handler must return a `Response` (or a promise of one) — it's wrapped in `Promise.resolve()` internally

## Dependency Injection

Any class can be a provider; extend `Provider` to opt into `onInit()`/`onDestroy()` lifecycle hooks:

```ts
import { Provider } from '@rasenganjs/server';

export class UserService extends Provider {
  async findAll() { /* ... */ }
}
```

Constructor injection is auto-wired by parameter name — no decorators needed:

```ts
export class UserController extends Controller {
  constructor(private userService: UserService) {
    super();
  }
}
```

Explicit `ProviderDefinition` form covers values, aliasing, and disambiguation:

```ts
providers: [
  { provide: Logger, useClass: FileLogger, deps: [ConfigToken] }, // class alias + explicit deps
  { provide: 'CONFIG', useValue: { port: 3000 } },                // static value, string token
]
```

Rules:
- Resolution is singleton-per-token: the same instance is shared by every module that can see it
- Auto-wiring extracts constructor parameter **names** via source inspection — an unregistered class referenced by name is auto-registered as a private provider owned by the resolving module (this is how controllers work without being listed in `providers`)
- A provider registered in one module is invisible to other modules unless it's in that module's `exports` (and the requesting module `imports` it, or the owning module is `global: true`) — otherwise resolution throws a directed "not visible here" error
- String-token resolution (`deps: ['CONFIG']`) matches case-insensitively against string keys or class names
- Every declared provider is eagerly constructed at boot (so `onInit()` fires and wiring mistakes fail fast), not lazily on first request

## Configuration & CLI

```ts
// rasengan.server.js
import { defineConfig } from '@rasenganjs/server';

export default defineConfig({
  entry: 'src/main.ts',
  port: 3006,
  watchDir: 'src/',
  preset: 'node', // 'node' | 'bun' | 'workerd'
  build: {
    outDir: '.rasengan',
    formats: ['directory'], // or ['single-file'], or both
    minify: false,
  },
});
```

```bash
rasengan-server dev --port 4000     # dev server with file watching (tsx watch)
rasengan-server build --preset node # bundle for production (esbuild)
rasengan-server start                # run the built output
```

Rules:
- Config resolution priority: defaults < `rasengan.server.js` file < CLI flags
- `entry` defaults to `src/main.ts`, `port` to `3000`, `host` to `0.0.0.0`, `watchDir` to `src/`
- `build.formats` controls output shape: `'single-file'` → one `server.bundle.mjs`; `'directory'` → one `.mjs` per source file preserving structure
