---
name: rasengan-server-validation
description: Schema validation patterns for @rasenganjs/server using @rasenganjs/validators. Covers zodAdapter, app.configureValidation({ adapter, onError }), per-route schemas (router.post(path, handler, { body, params, query })), controller-level schemas dictionaries matched by handler.name, the arrow-function/bound-method name-loss gotcha, schema precedence (per-route overrides controller-level), and custom validation error handlers. Use when adding request validation to a controller's routes.
license: MIT
metadata:
  author: Dilane Kombou
  framework: rasengan server
  version: "1.0.0-beta.0"
---

# @rasenganjs/server Validation Patterns

## When to Activate

- Adding `body`/`params`/`query` validation to a route
- Deciding between per-route schemas and a controller-level `schemas` dictionary
- Debugging why a controller-level schema isn't being applied to a handler
- Customizing the validation error response shape or status code
- Swapping the default Zod adapter for another schema library

## Configuring the Adapter

`@rasenganjs/server` defaults to `zodAdapter` with a built-in 400 JSON error handler. Override either via `app.configureValidation()`:

```ts
// src/main.ts
import { bootstrap } from '@rasenganjs/server';
import { zodAdapter } from '@rasenganjs/validators';
import appModule from './app.module';

bootstrap(async (app) => {
  app.registerModule(appModule);

  app.configureValidation({
    adapter: zodAdapter,
  });
});
```

Rules:
- `configureValidation()` merges with the defaults — pass only the field you want to change (`adapter` and/or `onError`)
- `@rasenganjs/validators` is a required dependency of `@rasenganjs/server`, not optional
- The default adapter requires Zod to be installed; a custom `SchemaAdapter` (`{ parse(schema, data), infer(schema) }`) can back any schema library (Valibot, ArkType, ...)

## Per-Route Schemas

Pass a `SchemaDefinition` (`{ body?, params?, query?, onError? }`) as the **last** argument to any `Router` verb method:

```ts
import { Controller, type RouteHandler, type Router } from '@rasenganjs/server';
import z from 'zod';

const UserParamSchema = z.object({
  id: z.coerce.number({ message: 'must be number' }),
});
const UserQuerySchema = z.object({
  page: z.coerce.number({ message: 'must be number' }).optional(),
  limit: z.coerce.number({ message: 'must be number' }).optional(),
});

export class UserController extends Controller {
  routes(router: Router) {
    router.get(
      '/:id',
      this.findOne,
      { params: UserParamSchema, query: UserQuerySchema }
    );
    router.post('/', this.create, { body: z.object({ name: z.string() }) });
  }

  findOne: RouteHandler = async (ctx) => {
    // ctx.params.id is a number here (coerced + validated)
    return ctx.res.json({ id: ctx.params.id });
  };

  create: RouteHandler = async (ctx) => {
    return ctx.res.json(ctx.body); // ctx.body typed as { name: string }
  };
}
```

To get `ctx.body`/`ctx.params`/`ctx.query` typed from the schema, annotate the handler's type with `RouteHandler<typeof schema>`:

```ts
create: RouteHandler<{ body: typeof CreateUserSchema }> = async (ctx) => {
  ctx.body.name; // typed as string
};
```

## Controller-Level Schemas

Declare a `schemas` dictionary keyed by **method name**; it's matched automatically against handlers registered without a per-route schema:

```ts
import { Controller, type RouteHandler, type Router } from '@rasenganjs/server';
import { UserService } from './user.service';
import z from 'zod';

const UserParamSchema = z.object({
  id: z.coerce.number({ message: 'must be number' }),
});
const UserQuerySchema = z.object({
  page: z.coerce.number({ message: 'must be number' }).optional(),
  limit: z.coerce.number({ message: 'must be number' }).optional(),
});

export class UserController extends Controller {
  schemas = {
    findOne: { params: UserParamSchema, query: UserQuerySchema },
    findAll: { query: UserQuerySchema },
    create: { body: z.object({ name: z.string() }) },
  } as const;

  constructor(private userService: UserService) {
    super();
  }

  routes(router: Router) {
    router.get('/', this.findAll);
    router.get('/:id', this.findOne, this.schemas.findOne);
    router.post('/', this.create, this.schemas.create);
  }

  findAll: RouteHandler = async (ctx) => {
    const list = await this.userService.findAll();
    return ctx.res.json(list);
  };

  findOne: RouteHandler<typeof this.schemas.findOne> = async (ctx) => {
    const user = await this.userService.findById(ctx.params.id);
    if (!user) return ctx.res.status(404).json({ error: 'User not found' });
    return ctx.res.json(user);
  };

  create: RouteHandler<typeof this.schemas.create> = async (ctx) => {
    const user = await this.userService.create(ctx.body);
    return ctx.res.json(user);
  };
}
```

Rules:
- Matching is done by `handler.name` at route-registration time — pass `this.findOne` (a class method reference) as the handler, **not** an inline arrow function, for name-based matching to work
- **Gotcha:** arrow functions assigned to class fields (`findOne: RouteHandler = async (ctx) => {...}`) still have a `.name` (the field name, e.g. `"findOne"`), so the pattern above works — but a genuinely anonymous or reassigned/bound function (`this.findOne.bind(this)`, an inline lambda passed directly to `router.get`) loses or changes its `.name` and won't match. For those, pass the schema explicitly as the route's last argument instead of relying on `schemas`
- **Precedence:** a per-route schema argument always overrides a controller-level `schemas` entry of the same handler name — pass both only when you intend the per-route one to win
- `this.schemas.findOne` as the third argument and the `schemas` dictionary matching are not mutually exclusive — passing it explicitly is redundant but harmless; omitting it relies purely on name matching

## Custom Validation Error Handler

Global, via `configureValidation`:

```ts
app.configureValidation({
  onError: (errors, ctx) =>
    Response.json({ code: 'VALIDATION_ERROR', errors }, { status: 422 }),
});
```

Per-route, via the schema's `onError` field (overrides the global handler for that route only):

```ts
router.post('/users', handler, {
  body: CreateUserSchema,
  onError: (errors, ctx) =>
    Response.json({ code: 'INVALID_USER', errors }, { status: 422 }),
});
```

`errors` is `ValidationError[]`, each `{ path: (string | number)[], message: string, code?: string }`. The default handler returns `Response.json({ errors }, { status: 400 })`.

Rules:
- Validation middleware runs after route-level middleware and before the handler, injected automatically at route-registration time whenever a schema is resolved (per-route or controller-level)
- A route with no schema at all skips validation entirely — no perf cost
