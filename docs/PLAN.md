# Zap Development Plan

## Vision

Zap combines the best ideas from multiple paradigms:

- **Erlang/BEAM**: Lightweight processes, message passing, fair scheduling, fault isolation
- **Pony**: Compile-time verified message types, no data races by construction
- **Zig**: Zero-cost abstractions via comptime, explicit error handling, no hidden control flow

Modern deployment (Kubernetes, blue-green deploys, rolling updates) handles what BEAM's hot code loading solved. Zap focuses on compile-time safety and simple deployment instead.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Build Pipeline                              │
├─────────────────────────────────────────────────────────────────────┤
│  Source (.zap) + zap.toml                                           │
│       ↓                                                             │
│  Compiler (type checking, dependency resolution)                    │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Executable (.zpc)          OR      Library (.zpl)          │   │
│  │  - All deps bundled                 - Bytecode + types      │   │
│  │  - Single file deployment           - Signed for security   │   │
│  │  - Run directly                     - Publishable           │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                           Zap Runtime                               │
├─────────────────────────────────────────────────────────────────────┤
│  Process Layer                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │ Process A   │  │ Process B   │  │ Process C   │                 │
│  │ ─────────── │  │ ─────────── │  │ ─────────── │                 │
│  │ Mailbox     │  │ Mailbox     │  │ Mailbox     │                 │
│  │ (typed)     │  │ (typed)     │  │ (typed)     │                 │
│  │ Heap        │  │ Heap        │  │ Heap        │                 │
│  │ (arena)     │  │ (arena)     │  │ (arena)     │                 │
│  └─────────────┘  └─────────────┘  └─────────────┘                 │
├─────────────────────────────────────────────────────────────────────┤
│  Scheduler (reduction-based, fair, no starvation)                   │
├─────────────────────────────────────────────────────────────────────┤
│  Distribution Layer (cluster, typed serialization)                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Development Phases

### Phase 1: Minimal Viable Scheduler

**Goal:** Prove we can build a fair, preemptive scheduler in Zig

**Deliverables:**
- [ ] `Process` struct with state, mailbox, reduction counter
- [ ] Round-robin scheduler with reduction-based preemption
- [ ] Basic message passing between processes
- [ ] Benchmark: 10,000 processes running without crashes
- [ ] Demonstrate preemption (long-running process yields fairly)

**Success Criteria:**
- Can spawn 10,000+ processes
- Fair scheduling (no process starves)
- Processes can send/receive typed messages
- Preemption works (infinite loop doesn't block others)

**Key Files:**
```
src/
├── process.zig      # Process struct, state machine
├── mailbox.zig      # Message queue per process
├── scheduler.zig    # Round-robin with reductions
└── main.zig         # Demo/benchmark
```

### Phase 2: Typed Protocols

**Goal:** Prove compile-time message type verification works

**Deliverables:**
- [ ] Protocol definition syntax using Zig comptime
- [ ] Compile-time verification that sends match receives
- [ ] Type-safe mailboxes that reject wrong message types
- [ ] Clear compile errors for protocol violations

**Key Innovation:** This is where Zap differentiates from Gleam. If typed protocols feel natural and catch real errors, we have something unique.

### Phase 3: Build Artifacts & Package System

**Goal:** Single-file deployment and secure package distribution

**Deliverables:**
- [ ] Executable artifact (.zpc) - all deps bundled into one file
- [ ] Library artifact (.zpl) - publishable package format
- [ ] `zap.toml` manifest with dependency declarations
- [ ] Lock file for reproducible builds
- [ ] Code signing for libraries
- [ ] Signature verification during build

**Artifact Types:**
```bash
zap build              # outputs {name}.zpc from zap.toml
zap pack               # outputs {name}-{version}.zpl (signed)
zap run app.zpc        # run executable
```

**Manifest Example:**
```toml
[package]
name = "myapp"
version = "1.0.0"
main = "src/main.zap"

[dependencies]
json = { version = "2.0.0", source = "github:zap-lang/json" }
http = { version = "1.5.0", source = "github:zap-lang/http" }
```

Libraries can have their own dependencies with different versions - resolved at compile time, not runtime.

### Phase 4: Simple Bytecode

**Goal:** Move from hand-written Zig to something more dynamic

**Deliverables:**
- [ ] Minimal bytecode instruction set
- [ ] Bytecode interpreter
- [ ] Basic compiler (simplified syntax → bytecode)
- [ ] Retain all type safety through bytecode

### Phase 5: Production Runtime

**Goal:** Run real applications on Zap

**Deliverables:**
- [ ] TCP/HTTP networking
- [ ] File I/O
- [ ] Timers and scheduling
- [ ] Supervision trees (basic)
- [ ] Logging and observability

### Phase 6: Language & Ecosystem

**Goal:** Full developer experience

**Deliverables:**
- [ ] Full language syntax design
- [ ] Compiler frontend
- [ ] Standard library
- [ ] Package manager (fetch from GitHub releases, verify signatures)
- [ ] LSP server
- [ ] Documentation

---

## Build in Public Strategy

### Content Pillars

**1. Weekly Progress (Twitter/X)**
- Screenshots of working code
- Benchmark results
- "Today I learned" moments
- Honest failures and pivots

**2. Technical Blog Series**
- "Why I'm building a BEAM alternative in 2025"
- "How Erlang's scheduler actually works"
- "10,000 processes in Zig: First benchmarks"
- "Typed message passing: What Gleam doesn't do"
- "Per-process versioning: A new approach to hot code loading"

**3. Monthly Deep Dives**
- BEAM internals explained
- Zig comptime tricks
- Comparisons with Lunatic/Firefly/Gleam

### Production Target

**Goal:** Run Zap in production within 3-4 months

**Candidate Projects:**
| Project | Why it's good | Complexity |
|---------|---------------|------------|
| Discord bot | Concurrent connections, message passing | Low |
| Webhook processor | Fault isolation, retry logic | Low-Medium |
| Real-time chat | Classic BEAM demo | Medium |
| Simple game server | Actor model showcase | Medium |

**Recommended first deployment:** Discord bot or webhook processor

### Funding Path

1. **GitHub Sponsors** - Start early, validates interest
2. **Grants:**
   - Sovereign Tech Fund
   - NLnet Foundation
   - GitHub Accelerator
3. **YC/VC** - If significant traction

**To be fundable:**
- Working software (not just ideas)
- Adoption metrics (stars, sponsors, users)
- Clear differentiation story
- Explanation of why this attempt succeeds where others failed

---

## Competitive Landscape

| Project | Status | Relationship to Zap |
|---------|--------|---------------------|
| **Gleam** | Active, v1.0 | Static types on BEAM, ML syntax. Zap is C-style, own runtime |
| **Lunatic** | Stalled (2023) | Similar goals, Rust+WASM. Study their architecture |
| **Firefly** | Archived (2024) | BEAM reimplementation. Learn from their failure |
| **BEAM** | Gold standard | Our inspiration, not our competition |

**Zap's Unique Position:**
- C-style syntax (vs Gleam's ML-style)
- Single-file deployment with bundled deps
- Signed packages for security
- Full AOT, no JIT pauses
- Not dependent on BEAM or WASM

---

## Resources

### BEAM Internals
- [The BEAM Book](https://blog.stenmans.org/theBeamBook/)
- [Erlang Scheduler Deep Dive](https://blog.appsignal.com/2024/04/23/deep-diving-into-the-erlang-scheduler.html)

### Similar Projects
- [Lunatic source](https://github.com/lunatic-solutions/lunatic)
- [Backstage (Zig actors)](https://github.com/Thomvanoorschot/backstage)

### Zig
- [Zig Comptime](https://kristoff.it/blog/what-is-zig-comptime/)
- [Zig Metaprogramming](https://ikrima.dev/dev-notes/zig/zig-metaprogramming/)

---

## Core Design Decisions

### Process Structure

```zig
const Process = struct {
    id: ProcessId,
    mailbox: TypedMailbox,
    heap: ArenaAllocator,
    reductions_remaining: u32,
    state: enum { ready, running, blocked, dead },
};
```

### Reduction-Based Scheduling

```
Process A: ████████░░░░░░░░ (4000 reductions)
                    ↓ yield
Process B: ████████░░░░░░░░ (4000 reductions)
                    ↓ yield
Process C: ████████░░░░░░░░ (4000 reductions)
                    ↓ yield
Process A: (continues)
```

Reductions counted at:
- Function calls
- Loop iterations
- Message send/receive
- Allocation beyond threshold

### Message Passing

Messages are **copied** between processes (no shared mutable state):

```zig
fn send(self: *Process, target: *Process, msg: anytype) !void {
    const copied = try target.heap.deepCopy(msg);
    try target.mailbox.push(copied);
}
```

### Error Handling

All errors explicit via error unions. No exceptions.

```zig
fn handleRequest(req: Request) !Response {
    const conn = try connect(req.url) orelse return error.ConnectionFailed;
    defer conn.close();
    return try conn.send(req);
}
```
