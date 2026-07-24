---
name: rasengan-server-drizzle
description: Drizzle ORM integration for @rasenganjs/server using @rasenganjs/drizzle. Covers DrizzleModule.forRoot({ adapter, connection, schema, global? }), the DataSource<TDb> Provider (constructor-injected by name like TypeORM's DataSource), the DrizzleAdapter<TConfig,TSchema,TDb> driver interface, the built-in nodePostgresAdapter (@rasenganjs/drizzle/drivers/node-postgres), the standalone runMigrations() script runner, and the forRoot()-throws-if-called-twice / DataSource-resolved-before-forRoot() constraints. Use when adding a Drizzle-backed database to a Rasengan server app or writing a Provider/Controller/Gateway/Queue that reads or writes the database.
license: MIT
metadata:
  author: Dilane Kombou
  framework: rasengan server
  version: "1.0.0-beta.0"
---

# @rasenganjs/server Drizzle ORM Patterns

## When to Activate

- Wiring a Postgres (or other Drizzle-supported) database into a Rasengan server app for the first time
- Writing a `Provider`, `Controller`, `Gateway`, or `Queue` that needs to read/write the database
- Writing or running a standalone migration script outside the running server
- Adding a new Drizzle driver/dialect beyond the built-in `node-postgres` one
- Debugging a `DataSource resolved before DrizzleModule.forRoot() ran` or `forRoot() was called more than once` error

## Wiring It Up

`DrizzleModule.forRoot({...})` connects **eagerly** (not lazily on first `.db` access) and returns a `ModuleConfig` you import like any other module. Call it exactly once per process, at module-definition time — before `bootstrap()` registers anything that injects `DataSource`:

```ts
// db.module.ts
import { defineModule, Provider } from '@rasenganjs/server';
import { pgTable, uuid, text } from 'drizzle-orm/pg-core';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { DrizzleModule, DataSource as GenericDataSource } from '@rasenganjs/drizzle';
import { nodePostgresAdapter } from '@rasenganjs/drizzle/drivers/node-postgres';

// 1. Your app's schema — @rasenganjs/drizzle never sees this.
const users = pgTable('users', {
  id: uuid('id').primaryKey(),
  email: text('email').notNull().unique(),
});
const schema = { users };

// 2. Pick a driver adapter + connection config. Swapping drivers later
//    is "change these two lines" — nothing else changes.
const adapter = nodePostgresAdapter<typeof schema>();
const connection = { connectionString: process.env.DATABASE_URL };

// 3. Configure the singleton ONCE.
export const dbModule = DrizzleModule.forRoot({ adapter, connection, schema });

// A local type alias pins DataSource's generic to THIS app's schema once,
// so every provider below writes bare `DataSource`, fully typed, zero
// type arguments — exactly like importing `DataSource` from `typeorm`.
export type DataSource = GenericDataSource<NodePgDatabase<typeof schema>>;
```

```ts
// app.module.ts
import { defineModule } from '@rasenganjs/server';
import { dbModule } from './db.module';
import { UserRepository } from './user.repository';

export default defineModule({
  imports: [dbModule /* , ...other feature modules */],
  providers: [UserRepository],
});
```

Rules:
- `DrizzleModule.forRoot()` registers `DataSource` as `global: true` by default — inject it from any module without adding `dbModule` to that module's own `imports`
- Calling `forRoot()` a second time in the same process throws — it's shared module-level connection state, not a per-call instance
- Export the local `type DataSource = GenericDataSource<NodePgDatabase<typeof schema>>` alias once (as shown) so every consumer writes plain `DataSource` fully typed — don't re-import `DataSource` from `@rasenganjs/drizzle` directly in app code

## Using DataSource in a Provider

```ts
// user.repository.ts
import { Provider } from '@rasenganjs/server';
import { eq } from 'drizzle-orm';
import { users } from './schema';
import type { DataSource } from './db.module';

export class UserRepository extends Provider {
  constructor(private readonly dataSource: DataSource) {
    super();
  }

  findByEmail(email: string) {
    return this.dataSource.db.select().from(users).where(eq(users.email, email));
  }
}
```

Rules:
- `DataSource` works exactly like any other DI-injected `Provider` — constructor-inject it into a `Controller`, `Gateway`, `Queue`, or another `Provider`
- **Do not rename or re-export `DataSource` under a different name.** rasengan-server's DI resolves constructor params by matching their name against a provider class's runtime `.name`; the exported class must stay literally named `DataSource`
- `dataSource.db` throws `DataSource resolved before DrizzleModule.forRoot() ran` if something resolves it before `forRoot()` has executed — make sure the module containing `DrizzleModule.forRoot(...)` (or a module importing it) is registered early in the app's module graph
- `DataSource.onDestroy()` closes the underlying connection automatically when the server shuts down — you never call `.close()` yourself in application code

## Driver Adapters

Only one driver ships in v1: `nodePostgresAdapter()` from the `@rasenganjs/drizzle/drivers/node-postgres` subpath (kept separate from the core `@rasenganjs/drizzle` export so importing the package never pulls in a specific driver's client library):

```ts
import { nodePostgresAdapter } from '@rasenganjs/drizzle/drivers/node-postgres';

const adapter = nodePostgresAdapter<typeof schema>();
// connection config is a pg.PoolConfig — e.g. { connectionString } or { host, port, user, password, database }
```

The `DrizzleAdapter<TConfig, TSchema, TDb>` interface is the whole extension point — adding a new driver/dialect is "write one file implementing this interface," no change needed anywhere else:

```ts
interface DrizzleAdapter<TConfig, TSchema extends Record<string, unknown>, TDb> {
  readonly name: string;
  connect(config: TConfig, schema: TSchema): {
    db: TDb;
    close(): Promise<void>;
    migrate(migrationsFolder: string): Promise<void>;
  };
}
```

Rules:
- `nodePostgresAdapter()`'s pool probes the connection once at connect time and kills the process (`SIGINT`) if the database is unreachable — a down DB fails loudly at boot instead of surfacing later as an opaque query failure
- The pool also registers an `error` listener so a dropped idle-client connection logs instead of crashing the process with an unhandled `'error'` event

## Running Migrations

Migration scripts run **outside** the server process, so they get their own short-lived connection instead of reusing `DrizzleModule`'s long-lived one — `runMigrations()` opens, migrates, and closes in one call:

```ts
// scripts/migrate.ts — run with e.g. `tsx scripts/migrate.ts`
import { runMigrations } from '@rasenganjs/drizzle';
import { nodePostgresAdapter } from '@rasenganjs/drizzle/drivers/node-postgres';
import { schema } from '../src/schema';

await runMigrations(
  nodePostgresAdapter<typeof schema>(),
  { connectionString: process.env.DATABASE_URL },
  schema,
  './drizzle/migrations'
);
```

Rules:
- Never call `runMigrations()` from inside the running server — it's a standalone-script API, deliberately not wired into `DrizzleModule.forRoot()`
- Generate the migrations folder itself with `drizzle-kit` (not part of this package) before running `runMigrations()` against it
