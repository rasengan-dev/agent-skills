---
name: rasengan-server-websockets
description: WebSocket patterns for @rasenganjs/server and @rasenganjs/ws. Covers the low-level app.websocket(path, { open, message, close, error }) API from @rasenganjs/server itself, and the higher-level Gateway abstract class (path, onConnect/onDisconnect, messages(router), GatewayMessageHandler, client.emit/join/leave/to(room).emit/broadcast.emit, this.server.to(room).emit), registration via defineModule({ gateways: [...] }) + app.registerPlugin(createWsPlugin()), and constructor DI into gateways. Use when adding real-time/WebSocket features to a Rasengan server app.
license: MIT
metadata:
  author: Dilane Kombou
  framework: rasengan server
  version: "1.0.0-beta.0"
---

# @rasenganjs/server WebSocket Patterns

## When to Activate

- Adding a raw WebSocket endpoint directly on `ServerApp`
- Building a room-based, event-driven WebSocket feature (chat, presence, live boards)
- Deciding between the low-level `app.websocket()` API and a `@rasenganjs/ws` `Gateway`
- Broadcasting to rooms or all connected clients, including from outside a connection handler (e.g. an HTTP controller)
- Injecting services into a Gateway via constructor DI

## Two Layers

`@rasenganjs/server` ships only a runtime-agnostic WebSocket route registry (`app.websocket()`) — no rooms, no event envelope, no broadcast helpers. `@rasenganjs/ws` is a `Controller`-equivalent convenience layer (`Gateway`) built entirely on top of it, adding rooms, a `{ event, data }` JSON envelope, DI, and broadcast scoping. Reach for a `Gateway` unless you need something `app.websocket()`'s three raw callbacks can't express.

## Low-Level: `app.websocket()`

```ts
// src/main.ts
bootstrap(async (app) => {
  app.websocket('/chat', {
    open(ctx) {
      console.log('[ws] client connected:', ctx.request.url);
    },
    message(ctx, data) {
      console.log('[ws] received:', data);
      ctx.socket.send(`echo: ${data}`);
    },
    close(_ctx, code, reason) {
      console.log('[ws] client disconnected:', code, reason);
    },
    error(_ctx, error) {
      console.error('[ws] connection error:', error);
    },
  });
});
```

Rules:
- Only static paths are supported (no `/chat/:room` dynamic segments)
- `ctx.socket` is the runtime-agnostic `WebSocketConnection`; `ctx.request` is the original upgrade `Request` (read query params/headers/cookies from it)
- `message(ctx, data)` receives raw frames (`string | ArrayBuffer`) — there is no built-in event envelope at this layer
- This registry is currently consumed by the Node adapter; other adapters (Bun, workerd) may not wire it up yet

## High-Level: `Gateway` (`@rasenganjs/ws`)

A `Gateway` is the `Controller` equivalent for WebSocket routes — declared in `defineModule({ gateways: [...] })`, resolved through the same DI container, registered via a `ModulePlugin`:

```ts
// src/main.ts
import { bootstrap } from '@rasenganjs/server';
import { createWsPlugin } from '@rasenganjs/ws';
import appModule from './app.module';

bootstrap(async (app) => {
  // Must run before registerModule() picks up gateways: [...]
  app.registerPlugin(createWsPlugin());
  app.registerModule(appModule);
});
```

```ts
// chat-room.module.ts
import { defineModule } from '@rasenganjs/server';
import { ChatRoomGateway } from './chat-room.gateway';
import { ChatRoomService } from './chat-room.service';

export default defineModule({
  gateways: [ChatRoomGateway],
  providers: [ChatRoomService],
});
```

```ts
// chat-room.gateway.ts
import {
  Gateway,
  GatewayRouter,
  type GatewayClient,
  type GatewayMessageHandler,
} from '@rasenganjs/ws';
import { ChatRoomService } from './chat-room.service';

export class ChatRoomGateway extends Gateway {
  path = '/rooms';

  constructor(private chatRoomService: ChatRoomService) {
    super();
  }

  onConnect(client: GatewayClient) {
    const room = new URL(client.request.url).searchParams.get('room') ?? 'lobby';
    client.data.room = room;
    client.join(room);
    client.emit('connected', { id: client.id, room });
  }

  onDisconnect(client: GatewayClient) {
    console.log(`${client.id} disconnected from "${client.data.room}"`);
  }

  messages(router: GatewayRouter) {
    router.on('sendMessage', this.handleSendMessage);
    router.on('switchRoom', this.handleSwitchRoom);
  }

  handleSendMessage: GatewayMessageHandler<{ text: string }> = (client, data) => {
    const room = client.data.room as string;
    const count = this.chatRoomService.recordMessage(room);

    // Everyone else in the room gets the message...
    client.to(room).emit('newMessage', { text: data.text, from: client.id, count });
    // ...the sender gets a direct ack instead.
    client.emit('messageSent', { count });
  };

  handleSwitchRoom: GatewayMessageHandler<{ room: string }> = (client, data) => {
    const previousRoom = client.data.room as string;
    client.leave(previousRoom);
    client.join(data.room);
    client.data.room = data.room;
    client.emit('roomSwitched', { from: previousRoom, to: data.room });
  };
}
```

## Client / Server Broadcast API

`GatewayClient` (passed to every handler):

| Member | Behavior |
|--------|----------|
| `client.id` | Per-connection id (not stable across reconnects) |
| `client.request` | The original upgrade `Request` |
| `client.data` | Free-form per-connection state bag |
| `client.join(room)` / `.leave(room)` | Room membership (local, synchronous) |
| `client.rooms()` | Rooms this client currently belongs to |
| `client.emit(event, data)` | Send to this client only |
| `client.to(room).emit(event, data)` | Broadcast to a room, **excluding** the sender |
| `client.broadcast.emit(event, data)` | Broadcast to everyone on this gateway **except** the sender |
| `client.disconnect(code?, reason?)` | Close this client's connection |

`Gateway.server` (`GatewayServer`) — for broadcasting from outside any connection's context (an HTTP controller, a timer):

```ts
export class BoardGateway extends Gateway {
  path = '/board';

  messages(router: GatewayRouter) {
    router.on('sendMessage', this.handleSendMessage);
  }

  handleSendMessage: GatewayMessageHandler<{ text: string }> = (client, data) => {
    client.broadcast.emit('message', data); // everyone else
    client.emit('message', data);           // sender only
    client.to('board').emit('message', data); // a specific room
  };
}
```

```ts
// from anywhere else (e.g. an HTTP controller with the gateway injected):
this.boardGateway.server.to('board').emit('announcement', { text: 'Server restarting' });
this.boardGateway.server.emit('broadcast', { text: 'Hello everyone' }); // no room, no sender to exclude
```

Rules:
- `Gateway extends Provider` — constructor DI works exactly like a `Controller` (`constructor(private chatRoomService: ChatRoomService) { super(); }`), and `onInit()`/`onDestroy()` fire on the same lifecycle
- `client.to(room)` (excludes sender) and `server.to(room)` (no sender to exclude, since there is none) are different objects with the same `Broadcaster` shape (`{ emit(event, data) }`) — pick based on whether you're inside a client handler
- Messages are a custom `{ event, data }` JSON envelope — **not** Socket.IO wire-compatible
- Event names starting with `$` are reserved for the protocol (`$error`, `$ping`, `$pong`, `$ack`) — `router.on('$foo', ...)` throws
- Registering the same event name twice on one gateway's `messages(router)` throws
- `createWsPlugin({ adapter, heartbeat })` accepts a `GatewayAdapter` for cross-process broadcasting: the default `MemoryGatewayAdapter` is single-process; pass a `RedisGatewayAdapter` to scale horizontally
- `defineModule({ gateways: [...], exports: [...] })` can export a gateway so another module can inject and call its `.server`
