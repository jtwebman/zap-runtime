# Zap Development Plan

## Vision

Zap combines the best ideas from multiple paradigms:

- **Erlang/BEAM**: Lightweight processes, message passing, fair scheduling, fault isolation
- **Pony**: Compile-time verified message types, no data races by construction
- **npm/Node.js**: Per-process dependency versioning, multiple library versions coexisting
- **Zig**: Zero-cost abstractions via comptime, explicit error handling, no hidden control flow

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Zap Runtime                               │
├─────────────────────────────────────────────────────────────────────┤
│  Source (.zap)                                                      │
│       ↓                                                             │
│  Compiler (type checking via Zig comptime)                          │
│       ↓                                                             │
│  Typed Bytecode (.zpc) ─────→ Module Registry (versioned)           │
│       ↓                              ↓                              │
│  AOT Native Compiler          Code Cache (per-version)              │
│       ↓                              ↓                              │
│  Native Code + Yield Points ←───────┘                               │
├─────────────────────────────────────────────────────────────────────┤
│  Process Layer                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │ Process A   │  │ Process B   │  │ Process C   │                 │
│  │ ─────────── │  │ ─────────── │  │ ─────────── │                 │
│  │ Mailbox     │  │ Mailbox     │  │ Mailbox     │                 │
│  │ (typed)     │  │ (typed)     │  │ (typed)     │                 │
│  │ Heap        │  │ Heap        │  │ Heap        │                 │
│  │ (arena)     │  │ (arena)     │  │ (arena)     │                 │
│  │ CodeBindings│  │ CodeBindings│  │ CodeBindings│                 │
│  │ {lib@1.0}   │  │ {lib@2.0}   │  │ {lib@1.0}   │                 │
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

### Phase 3: Per-Process Versioning

**Goal:** Prove our most novel idea works

**Deliverables:**
- [ ] Module registry with version tracking
- [ ] Per-process code bindings
- [ ] Spawn with version requirements
- [ ] Hot load new versions (existing processes unaffected)
- [ ] Demo: Two processes using different versions of same library

**This is the killer feature.** If it works well, it's a compelling story for adoption.

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
- [ ] Package manager
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
- Per-process versioning (novel)
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
    code_bindings: CodeBindings,
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
