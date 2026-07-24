---
name: rasengan-server-queues
description: Background job queue patterns for @rasenganjs/server using @rasenganjs/queue. Covers the Queue abstract class (name, onInit, jobs(router)), JobRouter.process(jobName, handler, { attempts, backoff, concurrency }), Queue.add(jobName, data, options) including { repeat: { every } } for recurring jobs and jobKey idempotency, registration via defineModule({ queues: [...] }) + app.registerPlugin(createQueuePlugin()), MemoryQueueAdapter vs RedisQueueAdapter, and constructor DI into queues (including injecting a gateway or service). Use when adding background/async job processing to a Rasengan server app.
license: MIT
metadata:
  author: Dilane Kombou
  framework: rasengan server
  version: "1.0.0-beta.0"
---

# @rasenganjs/server Queue Patterns

## When to Activate

- Adding background/async job processing (emails, digests, cleanup tasks) to a Rasengan server app
- Building a recurring/scheduled job (e.g. a periodic tick, cron-like digest)
- Triggering a job from an HTTP route and confirming it's queued
- Choosing between `MemoryQueueAdapter` (dev) and `RedisQueueAdapter` (persistent, multi-process)
- Understanding retry/backoff/dead-letter semantics for a job handler

## Registration

`Queue` is the `Controller` equivalent for background work — declared in `defineModule({ queues: [...] })`, resolved through the same DI container, registered via a `ModulePlugin`:

```ts
// src/main.ts
import { bootstrap } from '@rasenganjs/server';
import { createQueuePlugin } from '@rasenganjs/queue';
import appModule from './app.module';

bootstrap(async (app) => {
  // Must run before registerModule() picks up queues: [...].
  // No `adapter` option -> defaults to MemoryQueueAdapter (dev only, no Redis required).
  app.registerPlugin(createQueuePlugin());
  app.registerModule(appModule);
});
```

```ts
// queue.module.ts
import { defineModule } from '@rasenganjs/server';
import { HelloQueue } from './hello.queue';
import { QueueController } from './queue.controller';

export default defineModule({
  queues: [HelloQueue],
  controllers: [QueueController],
});
```

## Defining a Queue

```ts
// hello.queue.ts
import { Queue, JobRouter, type JobHandler } from '@rasenganjs/queue';

export class HelloQueue extends Queue {
  name = 'hello';

  // Runs once at boot — safe to call every time the process starts,
  // repeat registration is idempotent by jobKey (see below).
  async onInit() {
    await this.add('tick', {}, { repeat: { every: 1_000 } });
  }

  jobs(router: JobRouter) {
    router.process('greet', this.greet);
    router.process('tick', this.tick);
  }

  greet: JobHandler<{ name: string }> = async (job) => {
    console.log(`greeted ${job.data.name} (job ${job.id})`);
  };

  tick: JobHandler = async () => {
    console.log(`tick ${new Date().toISOString()}`);
  };
}
```

Triggering a job from an HTTP controller (`Queue` can be constructor-injected exactly like any provider):

```ts
// queue.controller.ts
import { Controller, type RouteHandler, type Router } from '@rasenganjs/server';
import z from 'zod';
import { HelloQueue } from './hello.queue';

export class QueueController extends Controller {
  schemas = {
    trigger: { body: z.object({ name: z.string() }) },
  } as const;

  constructor(private helloQueue: HelloQueue) {
    super();
  }

  routes(router: Router) {
    router.post('/jobs/hello', this.trigger, this.schemas.trigger);
  }

  trigger: RouteHandler<typeof this.schemas.trigger> = async (ctx) => {
    const jobId = await this.helloQueue.add('greet', { name: ctx.body.name });
    return ctx.res.json({ queued: true, jobId });
  };
}
```

## `.add()` Options

```ts
// One-shot
await queue.add('greet', { name: 'dilane' });

// Delayed — not reservable until now + delay ms
await queue.add('greet', { name: 'dilane' }, { delay: 60_000 });

// Recurring — idempotent by jobKey (derived from name+data, or an
// explicit repeat.key); calling this again with the same spec every
// boot does NOT create a duplicate schedule.
await queue.add('tick', {}, { repeat: { every: 1_000 } });
await queue.add('digest', { tenantId }, { repeat: { every: 86_400_000, key: `digest:${tenantId}` } });
```

`JobRouter.process(name, handler, options?)`:

```ts
router.process('welcome', this.sendWelcome, {
  attempts: 3,      // total attempts before dead-letter (default 1, no retry)
  backoff: 5_000,    // base retry delay ms, doubles per attempt (default 0)
  concurrency: 2,    // max in-flight handler calls for this job name (default 1)
});
```

Rules:
- `.add()` resolves with the job's `id` — except for a `{ repeat }` registration, where it resolves with the recurring job's stable `jobKey` instead
- `delay` and `repeat` are mutually exclusive on the same `.add()` call
- A handler resolving = job complete; throwing = retry with exponential backoff (`backoff * 2^(attempt-1)`) until `attempts` is exhausted, then the job moves to the dead-letter list
- Handlers must be idempotent — this is at-least-once delivery, not exactly-once
- Inspect/recover dead jobs with `queue.getDead()` and `queue.retryDead(id)` (resets `attempt` to 1 and re-queues)
- Registering the same job name twice on one queue's `jobs(router)` throws

## Adapters

```ts
// Default — nothing to configure, dev only, jobs are lost on process restart
app.registerPlugin(createQueuePlugin());
```

```ts
// Persistent, multi-process safe
import { createQueuePlugin, RedisQueueAdapter } from '@rasenganjs/queue';
import Redis from 'ioredis';

const client = new Redis();
const adapter = new RedisQueueAdapter({
  client,
  blockingClient: client.duplicate(), // reserve()'s BLMOVE needs its own connection
});

app.registerPlugin(createQueuePlugin({ adapter }));
```

`createQueuePlugin(options)`:

| Option | Default | Purpose |
|--------|---------|---------|
| `adapter` | `MemoryQueueAdapter` | Job storage — swap for `RedisQueueAdapter` to persist |
| `worker` | `true` | Set `false` for a produce-only process (`.add()` still works, nothing is reserved/processed here) |
| `stallTimeout` | `30_000` ms | How long a reserved job may go unacknowledged before the sweeper reclaims it |
| `sweepInterval` | `5_000` ms | How often the sweeper promotes due delayed/repeat jobs and reclaims stalled ones |

Rules:
- `Queue extends Provider` — constructor DI works exactly like a `Controller` or `Gateway` (`constructor(private mailer: MailerService) { super(); }`), including injecting one queue into another, or a `@rasenganjs/ws` `Gateway` into a queue for job-triggered broadcasts
- `defineModule({ queues: [...], exports: [...] })` can export a queue so another module can inject the same instance and call `.add()`
- `RedisQueueAdapter` requires `ioredis` as a dependency — it's not bundled by default
