# Zap

A statically-typed, functional programming language with BEAM-inspired concurrency - lightweight processes, message passing, and fault isolation.

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

| Feature          | BEAM                        | Zap                                       |
| ---------------- | --------------------------- | ----------------------------------------- |
| Type system      | Dynamic                     | Fully static, compile-time verified       |
| Message types    | Any term                    | Typed protocols, verified at compile time |
| Error handling   | Exceptions + "let it crash" | Error unions, explicit propagation        |
| Module versions  | 2 (current + old)           | Unlimited, per-process binding            |
| JIT pauses       | Yes (with JIT)              | No - AOT compiled                         |
| Hot code loading | Global swap                 | Per-process version selection             |

## Example

```typescript
// Define message and response types
// 'type' keyword automatically adds __type field from the type name

// Messages the User actor receives
type GetUsername = {};
type UserJoined = { username: string };
type UserLeft = { username: string };
type NewMessage = { username: string; content: string };

// Messages the ChatRoom actor receives
type JoinMsg = { username: string; process: User };
type LeaveMsg = { username: string };
type ChatMsg = { username: string; content: string };

// Protocol = union of Cast<Msg> and Call<Msg, Response>
// Cast = fire and forget, Call = request/response
type UserProtocol =
  | Cast<UserJoined>
  | Cast<UserLeft>
  | Cast<NewMessage>
  | Call<GetUsername, string>;

type ChatRoomProtocol =
  | Cast<JoinMsg>
  | Cast<LeaveMsg>
  | Cast<ChatMsg>;

// Define actors by their protocol
const User = actor<UserProtocol>();
const ChatRoom = actor<ChatRoomProtocol>();

// Spawn processes
const room = spawn(ChatRoom);
const me = spawn(User);

// Call waits for response (typed!)
const myUsername = me.call({ :GetUsername });  // type: string

// Cast fires and forgets
room.cast({ :JoinMsg, username: myUsername, process: me });
room.cast({ :ChatMsg, username: myUsername, content: "Hello!" });

room.cast({ :Invalid }); // COMPILE ERROR: Invalid is not in ChatRoomProtocol

// Actor state
type ChatRoomState = {
  users: Map<string, { process: User }>;
};

// Handle messages - pure functional with pattern matching
function chatRoom(ctx: Context<ChatRoomProtocol>, state: ChatRoomState = { users: Map.empty() }) {
  const msg = receive(ctx);

  // :TypeName is sugar for __type: "TypeName"
  match(msg, {
    { :JoinMsg, username, process }: () => {
      // Notify existing users
      state.users.forEach((user) =>
        user.process.cast({ :UserJoined, username })
      );
      return { ...state, users: state.users.set(username, { process }) };
    },

    { :ChatMsg, username, content }: () => {
      // Send to all other users
      state.users.forEach((user, key) => {
        if (key !== username) {
          user.process.cast({ :NewMessage, username, content });
        }
      });
      return state;
    },

    { :LeaveMsg, username }: () => {
      const users = state.users.delete(username);
      // Notify remaining users
      users.forEach((user) =>
        user.process.cast({ :UserLeft, username })
      );
      return { ...state, users };
    },
  });
}
```

## Language Design Choices

### Built-in Types

**Primitives:**
| Type | Description |
|------|-------------|
| `bool` | `true` or `false` |
| `int` | 64-bit signed integer (overflow is a runtime error) |
| `bigint` | Arbitrary precision integer |
| `float` | 64-bit IEEE 754 floating point |
| `decimal(p, s)` | Exact decimal with precision `p` and scale `s` (for money) |
| `string` | Immutable UTF-8 text |
| `bytes` | Immutable byte sequence (for binary data, I/O, networking) |

**Collections:**
| Type | Description |
|------|-------------|
| `Array<T>` | Immutable indexed collection |
| `Map<K, V>` | Immutable key-value collection |
| `Set<T>` | Immutable unique collection |

**Compound:**
| Type | Description |
|------|-------------|
| `{ ... }` | Record - bag of key-value pairs |
| `(T, U, V)` | Tuple - fixed-size, positional |
| `T?` | Optional - `Some(value)` or `None` - no null! |

```typescript
// Primitives
const count: int = 42;
const price: decimal(10, 2) = 19.99;
const name: string = "alice";

// Collections (immutable - methods return new collections)
const nums: Array<int> = [1, 2, 3];
const updated = nums.push(4);  // [1, 2, 3, 4] - new array

// Tuples - great for multiple returns
function divmod(a: int, b: int): (int, int) {
  return (a / b, a % b);
}
const (quotient, remainder) = divmod(10, 3);

// Optional with ? instead of null
function find(id: string): User? {
  // return Some(user) or None
}

match(find("123"), {
  { :Some, value }: () => console.log(value.name),
  { :None }: () => console.log("not found"),
});
```

### Types, not classes
No `class` keyword. Just `type` for data and `function` for behavior. Keeps things simple and functional.

```typescript
// Yes
type User = { name: string; age: int };
function createUser(name: string, age: int): User { ... }

// No classes
class User { constructor(...) { ... } }
```

### `type` only, no `interface`
Unlike TypeScript, there's no `interface` vs `type` confusion. Just use `type` for everything.

### Records and Types
`{}` is a record - just a bag of key-value pairs. Pattern matching works on records directly.

```typescript
// A plain record
const msg = { username: "alice", content: "hello" };

// Pattern match on any record
match(msg, {
  { username, content }: () => console.log(username, content),
});
```

`type` creates a named record that automatically includes a `__type` property. This is just sugar for easier pattern matching - under the hood, everything is records and pattern matching.

```typescript
type JoinMsg = { username: string };
// Equivalent to a record with: { __type: "JoinMsg", username: string }

// So you can pattern match on the __type
match(msg, {
  { __type: "JoinMsg", username }: () => ...,
  { __type: "LeaveMsg", username }: () => ...,
});
```

### `:TypeName` syntax
Shorthand for the `__type` field when creating values or pattern matching:

```typescript
// Creating a value
room.cast({ :JoinMsg, username: "alice" });
// Equivalent to: { __type: "JoinMsg", username: "alice" }

// Pattern matching
match(msg, {
  { :JoinMsg, username }: () => { ... },
  { :LeaveMsg, username }: () => { ... },
});
```

### Pattern matching on structure
`match()` matches on actual structure, not type names. Destructuring is built-in.

```typescript
match(value, {
  { :Some, value }: () => console.log(value),
  { :None }: () => console.log("nothing"),
});
```

### Protocols with `Cast<T>` and `Call<T, R>`
Actor protocols are unions of message types:
- `Cast<Msg>` - fire and forget
- `Call<Msg, Response>` - request/response (can include error types)

```typescript
type MyProtocol =
  | Cast<PingMsg>
  | Cast<NotifyMsg>
  | Call<GetDataMsg, Data | NotFoundError>;
```

### Typed process references
Spawned processes are typed by their protocol. You can only send messages they accept.

```typescript
const user: User = spawn(User);
user.cast({ :NewMessage, ... });  // OK - NewMessage is in UserProtocol
user.cast({ :JoinMsg, ... });     // COMPILE ERROR - JoinMsg not in UserProtocol
```

### Immutable data structures
Built-in immutable `Map`, `Set`, `List` with functional update methods:

```typescript
const users = Map.empty<string, User>();
const updated = users.set("alice", user);  // returns new Map
const removed = updated.delete("alice");   // returns new Map
```

### Explicit error handling
No exceptions. Use union types for errors:

```typescript
type Result = Data | NotFoundError | NotAuthorizedError;

match(result, {
  { :Data, value }: () => use(value),
  { :NotFoundError, message }: () => log(message),
  { :NotAuthorizedError, message }: () => deny(),
});
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

### Phase 5: JavaScript Target

Compile Zap to idiomatic JavaScript - not a runtime emulation, just clean JS:

```typescript
// Zap source
const room = spawn(ChatRoom);
room.cast({ :JoinMsg, username: "alice" });
const users = room.call({ :GetUsers });

// Compiles to JavaScript
const room = new ChatRoom();
room.joinMsg({ username: "alice" });  // Promise<void>
const users = await room.getUsers();  // Promise<UserList>
```

- [ ] Actors → Classes with async methods
- [ ] Functions → Functions
- [ ] Types → TypeScript definitions (.d.ts)
- [ ] Pattern matching → switch/if-else

**Type mapping to JavaScript:**
| Zap | JavaScript |
|-----|------------|
| `bool` | `boolean` |
| `int` | `number` (with overflow check → error) |
| `bigint` | `BigInt` |
| `float` | `number` |
| `decimal` | TBD (decimal.js or custom) |
| `string` | `string` |
| `bytes` | `Uint8Array` |
| `Array<T>` | `Array<T>` |
| `Map<K,V>` | `Map<K,V>` |
| `Set<T>` | `Set<T>` |
| `(T, U)` | `[T, U]` |
| `T?` | `T \| null` |

## Memory Model

Zap uses **per-process garbage collection** inspired by BEAM:

- **Isolated heaps** - Each process has its own heap, no shared memory
- **Per-process GC** - When GC runs, only that one process pauses (microseconds, not milliseconds)
- **Message copying** - Messages are copied between processes, no shared references
- **Instant cleanup** - When a process dies, its entire heap is freed at once

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Process A     │  │   Process B     │  │   Process C     │
│   ┌─────────┐   │  │   ┌─────────┐   │  │   ┌─────────┐   │
│   │  Heap   │   │  │   │  Heap   │   │  │   │  Heap   │   │
│   │ (64KB)  │   │  │   │ (128KB) │   │  │   │ (32KB)  │   │
│   └─────────┘   │  │   └─────────┘   │  │   └─────────┘   │
│   GC: 50μs      │  │   GC: 100μs     │  │   GC: 25μs      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
        ↑
    GC happens here,
    other processes
    keep running
```

**Why this works:**
- No global "stop the world" pauses
- Small heaps = fast GC (microseconds)
- Process isolation = independent GC scheduling
- Short-lived processes may never GC at all
- **GC yields like everything else** - even a large collection uses the same reduction-based scheduling, so it never blocks other processes

**Message passing:**
```typescript
// Messages are COPIED, not shared
processA.cast({ :Data, value: largeObject });
// processA and processB each have their own copy
// No shared references, no data races
```

## Implementation

The Zap compiler and runtime are written in **Zig** because:

- **Comptime** - Powerful compile-time metaprogramming for our type system
- **No hidden control flow** - Matches our "explicit everything" philosophy
- **C interop** - Can use existing C libraries seamlessly
- **Performance** - Zero-cost abstractions, predictable latency, no GC pauses

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
