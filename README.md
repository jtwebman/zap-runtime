# Zap

A BEAM-inspired runtime written in Zig with static typing, predictable latency, and per-process dependency versioning.

> **Status:** Early development. Follow along as we build this in public.

## Why Zap?

We love what Erlang/BEAM gives us:
- Lightweight processes (millions of them)
- Fair scheduling (no process starves)
- Fault isolation (one crash doesn't take down the system)
- Message passing (no shared mutable state)

But we want:
- **Static types** - Catch errors at compile time, not in production at 3am
- **Typed protocols** - Know exactly what messages an actor accepts
- **C-style syntax** - Familiar to most developers (not ML-style)
- **Per-process versioning** - Different processes can use different library versions
- **Predictable latency** - AOT compiled, no JIT pauses

## Key Differentiators

| Feature | BEAM | Zap |
|---------|------|-----|
| Type system | Dynamic | Fully static, compile-time verified |
| Message types | Any term | Typed protocols, verified at compile time |
| Error handling | Exceptions + "let it crash" | Error unions, explicit propagation |
| Module versions | 2 (current + old) | Unlimited, per-process binding |
| JIT pauses | Yes (with JIT) | No - AOT compiled |
| Hot code loading | Global swap | Per-process version selection |

## Example

```typescript
// Define message and response types
type JoinMsg = { userId: UserId; username: string };
type LeaveMsg = { userId: UserId };
type ChatMsg = { userId: UserId; content: string };
type GetUsersMsg = { };

type UserJoined = { userId: UserId; username: string };
type UserLeft = { userId: UserId };
type NewMessage = { userId: UserId; username: string; content: string };
type UserList = { users: Array<{ userId: UserId; username: string }> };

// Error types
type NotFoundError = { error: "NotFound"; message: string };
type NotAuthorizedError = { error: "NotAuthorized"; message: string };

// Protocol defines message -> response mapping
// void = fire and forget (cast), type = expects response (call)
type ChatRoomProtocol = {
  Join: void;                                  // cast - no response
  Leave: void;                                 // cast - no response
  Message: void;                               // cast - no response
  GetUsers: UserList | NotAuthorizedError;     // call - returns users or error
  GetUser: UserInfo | NotFoundError;           // call - returns user or not found
};

// Actor is defined purely by its protocol type
const ChatRoom = actor<ChatRoomProtocol>();

// Spawn a new process
const room = spawn(ChatRoom);

// Fire and forget (cast) - returns immediately
room.cast("Join", { userId: id, username: "alice" });
room.cast("Message", { userId: id, content: "Hello!" });

// Request/reply (call) - waits for response, fully typed
const users = room.call("GetUsers", {});  // type: UserList

room.cast("Invalid", {}); // COMPILE ERROR: 'Invalid' is not in ChatRoomProtocol

// Actor state
type ChatRoomState = {
  users: Record<UserId, { username: string }>;
};

// Handle messages - pure functional
function chatRoom(ctx: Context<ChatRoomProtocol>, state: ChatRoomState = { users: {} }) {
  const msg = receive(ctx);

  switch (msg.type) {
    case "Join": {
      const users = { ...state.users, [msg.userId]: { username: msg.username } };
      broadcast(ctx, { type: "UserJoined", userId: msg.userId, username: msg.username });
      return chatRoom(ctx, { ...state, users });
    }
    case "Message": {
      const user = state.users[msg.userId];
      if (user) {
        broadcast(ctx, { type: "NewMessage", userId: msg.userId, username: user.username, content: msg.content });
      }
      return chatRoom(ctx, state);
    }
    case "Leave": {
      const { [msg.userId]: _, ...users } = state.users;
      broadcast(ctx, { type: "UserLeft", userId: msg.userId });
      return chatRoom(ctx, { ...state, users });
    }
    case "GetUsers": {
      // reply() sends response back to caller
      reply(ctx, { users: Object.entries(state.users).map(([id, u]) => ({ userId: id, ...u })) });
      return chatRoom(ctx, state);
    }
  }
}
```

## Roadmap

### Phase 1: Core Runtime (In Progress)
- [ ] Process scheduler with reduction-based preemption
- [ ] Typed mailboxes and message passing
- [ ] Basic I/O (TCP, files)
- [ ] Run 10,000+ processes

### Phase 2: Type System
- [ ] Compile-time protocol verification
- [ ] Type-safe message serialization
- [ ] Error union integration

### Phase 3: Per-Process Versioning
- [ ] Module registry with versions
- [ ] Per-process code bindings
- [ ] Hot loading without affecting running processes

### Phase 4: Bytecode & Language
- [ ] Bytecode instruction set
- [ ] Interpreter
- [ ] Simple compiler frontend

## Why Zig?

- **Comptime** - Powerful compile-time metaprogramming for our type system
- **No hidden control flow** - Matches our "explicit everything" philosophy
- **C interop** - Can use existing C libraries seamlessly
- **Performance** - Zero-cost abstractions, no GC pauses

## Prior Art & Inspiration

Zap builds on ideas from:
- **[BEAM/Erlang](https://www.erlang.org/)** - The gold standard for actor-based concurrency
- **[Pony](https://www.ponylang.io/)** - Compile-time verified message types
- **[Gleam](https://gleam.run/)** - Proving there's demand for typed BEAM
- **[Lunatic](https://github.com/lunatic-solutions/lunatic)** - Erlang-inspired WASM runtime

## Building in Public

I'm documenting the entire journey of building Zap:
- **Twitter:** [@jtwebman](https://x.com/jtwebman)

## Contributing

Zap is in early development. If you're interested in:
- VM/runtime design
- Type system implementation
- Zig programming
- Testing and benchmarking

Open an issue or reach out!

## License

MIT OR Apache-2.0 (dual-licensed)
