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
// Define what messages an actor can receive and send
const ChatRoom = actor({
  receives: {
    Join: { userId: UserId, username: string },
    Leave: { userId: UserId },
    Message: { userId: UserId, content: string },
  },
  sends: {
    UserJoined: { userId: UserId, username: string },
    UserLeft: { userId: UserId },
    NewMessage: { userId: UserId, username: string, content: string },
  },
});

// Spawn a new process
const room = spawn(ChatRoom);

// Type-safe message sending
room.send({ type: "Join", userId: id, username: "alice" });
room.send({ type: "Message", userId: id, content: "Hello!" });

room.send({ type: "Invalid" }); // COMPILE ERROR: 'Invalid' is not a valid message type

// Handle messages in the actor
function chatRoom(ctx: Context<typeof ChatRoom>) {
  const users = new Map<UserId, User>();

  while (true) {
    const msg = ctx.receive();

    switch (msg.type) {
      case "Join":
        users.set(msg.userId, { username: msg.username });
        ctx.broadcast({ type: "UserJoined", userId: msg.userId, username: msg.username });
        break;
      case "Message":
        const user = users.get(msg.userId);
        if (user) {
          ctx.broadcast({ type: "NewMessage", userId: msg.userId, username: user.username, content: msg.content });
        }
        break;
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
