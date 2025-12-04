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
- **Single-file deployment** - Compile to one bundled executable
- **Signed packages** - Code signing for secure distribution
- **Predictable latency** - AOT compiled, no JIT pauses

## Key Differentiators

| Feature          | BEAM                        | Zap                                       |
| ---------------- | --------------------------- | ----------------------------------------- |
| Type system      | Dynamic                     | Fully static, compile-time verified       |
| Message types    | Any term                    | Typed protocols, verified at compile time |
| Error handling   | Exceptions + "let it crash" | Error unions, explicit propagation        |
| Deployment       | Release packages            | Single bundled executable (.zpc)          |
| Package security | Trust-based                 | Code signing with verification            |
| JIT pauses       | Yes (with JIT)              | No - AOT compiled                         |

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

### Process lifecycle and linking

Actors define both their protocol and exit reasons:

```typescript
// Default: only Normal exit (handle all errors internally)
const Worker = actor<WorkerProtocol>();
// Equivalent to: actor<WorkerProtocol, { :Normal }>

// Custom exit reasons for coordinated shutdown
type ChatRoomExit =
  | { :Normal }
  | { :Shutdown, reason: string }
  | { :RoomExpired, roomId: string };

const ChatRoom = actor<ChatRoomProtocol, ChatRoomExit>();
```

Link to a process to receive its exit notification:

```typescript
const room = spawn(ChatRoom, { link: self() });

// Tell it to shut down
room.cast({ :Shutdown });

// Parent receives typed exit
type RoomExited = Exited<ChatRoom>;  // derives from ChatRoomExit

match(msg, {
  { :RoomExited, id, reason }: () => {
    match(reason, {
      { :Normal }: () => log("room closed"),
      { :Shutdown, reason }: () => log("room shut down: " + reason),
      { :RoomExpired, roomId }: () => archiveRoom(roomId),
    });
  }
});
```

No magic crashes - memory allocation and spawning return `Error<T>` types that must be handled explicitly. Linking is for coordinated lifecycle management, not crash recovery.

### Immutable data structures
Built-in immutable `Map`, `Set`, `List` with functional update methods:

```typescript
const users = Map.empty<string, User>();
const updated = users.set("alice", user);  // returns new Map
const removed = updated.delete("alice");   // returns new Map
```

### Explicit error handling
No exceptions. Use union types for errors. Wrap error types in `Error<T>` to distinguish them from success types:

```typescript
// Define error types with Error<T> wrapper
type NotFoundError = Error<{ message: string }>;
type NotAuthorizedError = Error<{ message: string }>;

type Result = Data | NotFoundError | NotAuthorizedError;

match(result, {
  { :Data, value }: () => use(value),
  { :NotFoundError, message }: () => log(message),
  { :NotAuthorizedError, message }: () => deny(),
});
```

Use `try` for early return on errors (like Zig). Only `Error<T>` types propagate:

```typescript
type NotFoundError = Error<{ id: string }>;
type DbError = Error<{ code: int }>;

function getProfile(id: string): Profile | NotFoundError | DbError {
  const user = try getUser(id);        // returns early if Error<T>
  const prefs = try getPrefs(user.id); // returns early if Error<T>
  return { user, prefs };
}

// Multiple success types work fine - only Error<T> propagates
function parse(input: string): User | Guest | ParseError {
  // User and Guest are both valid returns
  // only ParseError (an Error<T>) propagates with try
}
```

The compiler enforces that all errors are handled at process boundaries - no unhandled errors can escape an actor's entry point.

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

### Phase 3: Build Artifacts & Packages

- [ ] Single-file executable (.zpc) with bundled deps
- [ ] Library package format (.zpl)
- [ ] Code signing and verification
- [ ] `zap.toml` manifest and lock files

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

## Standard Library

**Keywords (not functions):**
- `const`, `let`, `type`, `function`, `match`, `try`, `return`, `if`, `else`, `for`, `while`

**Built-in types (no import):**
- Primitives: `bool`, `int`, `bigint`, `float`, `decimal`, `string`, `bytes`
- Structural: `{ }` records, `(T, U)` tuples
- Special: `Error<T>`, `T?`

**Everything else is imported:**
```typescript
import { Array, Map, Set } from "collections"
import { spawn, cast, call, self, receive, link } from "actor"
import { print, panic } from "io"
import { read, write } from "fs"
import { listen, connect } from "net"
import { get, post } from "http"
import { now, sleep } from "time"
import { encode, decode } from "json"
import { hash, verify } from "crypto"
import { format } from "fmt"
import { env, args } from "os"
```

This makes every capability explicit - AI tools can learn the module catalog and the pattern is always the same.

**Generic functions:**
```typescript
function map<T, U>(arr: Array<T>, fn: (item: T) => U): Array<U> {
  // ...
}
```

## Control Flow

**Functional iteration (no `for` keyword):**
```typescript
items.each((item) => { ... })
items.map((item) => item.name)
items.filter((item) => item.active)
items.reduce(0, (acc, item) => acc + item.value)
items.eachWithIndex((item, i) => { ... })
users.each((key, value) => { ... })  // maps
```

**Imperative loops for stateful code:**
```typescript
while (condition) { }
do { } while (condition)

// break/continue work in while loops
while (true) {
  if (done) break;
  if (skip) continue;
}
```

**No recursion allowed:**
```typescript
// COMPILE ERROR: Recursion not allowed
function factorial(n: int): int {
  if (n <= 1) return 1;
  return n * factorial(n - 1);  // error: calls itself
}

// OK: Use while instead
function factorial(n: int): int {
  let result = 1;
  let i = n;
  while (i > 1) {
    result = result * i;
    i = i - 1;
  }
  return result;
}
```

The compiler detects both direct and mutual recursion (A calls B calls A). This makes resource usage predictable and prevents stack overflows. Use explicit data structures (stacks, queues) for tree-like algorithms.

## Mutability

**Variables can be `const` or `let`:**
```typescript
const x = 5;
x = 6;  // COMPILE ERROR: cannot rebind const

let y = 5;
y = 6;  // OK: let can be rebound
```

**All data structures are immutable.** `let` means the variable can point to a new value:
```typescript
let items = [1, 2, 3];
items = items.push(4);  // items now points to new array [1,2,3,4]
// original [1,2,3] is unchanged
```

No mutable data structures, no shared mutable state, no data races.

## Actor Lifecycle

Actors are pure functions. The runtime handles the message loop.

```typescript
// Define state type
type ChatRoomState = { users: Map<string, User> };

// Actor definition - links protocol to state type
const ChatRoom = actor<ChatRoomProtocol, ChatRoomState>();

// Pure handler function
function chatRoomHandler(state: ChatRoomState, msg: ChatRoomProtocol): ChatRoomState {
  match(msg, {
    { :JoinMsg, username, user }: () => ({ ...state, users: state.users.set(username, user) }),
  })
}

// Optional init and terminate functions
function initChatRoom(): ChatRoomState {
  return { users: Map.empty() };
}

function termChatRoom(state: ChatRoomState): void {
  saveToDb(state);
}

// Spawn variants
const room1 = spawn(chatRoomHandler, initChatRoom, termChatRoom);
const room2 = spawn(chatRoomHandler, initChatRoom);              // no terminate
const room3 = spawn(chatRoomHandler, { users: Map.empty() });    // direct state
```

**Spawn signature:**
```typescript
spawn<S>(
  handler: (state: S, msg: P) => S,
  init: S | () => S,
  terminate?: (state: S) => void
)
```

## Testing

Built-in test runner with functional test library:

```bash
zap test              # runs all *_test.zap files
zap test src/chat     # runs tests in specific path
```

```typescript
import { describe, it, expect, setup, singleton } from "test"

// Shared across all tests (one instance per test run)
const dbPool = singleton(
  () => createPool("postgres://..."),
  (pool) => pool.close()
);

// Fresh for each describe/it
const cache = setup(
  () => createCache(),
  (c) => c.clear()
);

describe("UserRepository", (dbPool), (pool) => {
  it("saves user", () => {
    const conn = pool.acquire();
    conn.save(user);
    conn.release();
  });
});

describe("CachedUserRepo", (dbPool, cache), (pool, cache) => {
  it("caches", () => {
    pool.acquire().save(user);
    cache.set(user.id, user);
  });
});

// No resources needed
describe("pure functions", () => {
  it("adds numbers", () => {
    expect(add(1, 2)).toBe(3);
  });
});
```

- `singleton()` - one instance for entire test run
- `setup()` - fresh instance per describe/it
- Tuple for multiple resources
- Actors are pure functions, so just call handlers directly and assert on state

## Operators

**Arithmetic (`int`, `bigint`, `float`, `decimal` - same type only):**

| Operator | Description |
|----------|-------------|
| `+` | add |
| `-` | subtract |
| `*` | multiply |
| `/` | divide |
| `%` | modulo |

```typescript
const a: int = 5 + 3;        // OK
const b: int = 5 + 3.0;      // COMPILE ERROR: int + float
const c: decimal(10,2) = 19.99 + 5.00;   // OK: same precision/scale
const d: decimal(10,4) = 1.0000 + c;     // COMPILE ERROR: decimal(10,4) + decimal(10,2)
```

**Comparison (same type only):**

| Operator | Description |
|----------|-------------|
| `==`, `!=` | equality |
| `<`, `>`, `<=`, `>=` | ordering |

**Logical (`bool` only):**

| Operator | Description |
|----------|-------------|
| `&&` | and |
| `\|\|` | or |
| `!` | not |

No custom operators. No cross-type operations. No currying. No pipe operator.

Keep it simple - use explicit variable bindings:

```typescript
const id = try getUserId();
const user = try fetchUser(db, id);
const validated = try validateUser(rules, user);
const saved = try saveUser(db, validated);
```

## Strings

No `+` for strings. Three options:

```typescript
import { format } from "fmt"
import { StringBuilder } from "str"

// Backticks for interpolation
const msg = `${name} has ${count} items`;

// Format for complex formatting
const report = format("{} - {.2f}%", label, percentage);

// StringBuilder for efficient building (one allocation)
const html = StringBuilder.new()
  .push("<div>")
  .push(content)
  .push("</div>")
  .join();

// With separator
const path = StringBuilder.new()
  .push(base)
  .push(folder)
  .push(file)
  .join("/");  // "base/folder/file"
```

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

MIT

By contributing to Zap, you agree that your contributions will be licensed under the MIT license.
