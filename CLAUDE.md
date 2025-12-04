# Zap Language

Statically-typed language with BEAM-style concurrency, C-like syntax, compiling to native (Zig) and JavaScript.

## Key Differentiators
- Typed actor protocols (compile-time message verification)
- Single-file deployment (`.zpc` bundles all deps)
- Signed packages (`.zpl`) for secure distribution
- AOT compilation (no JIT pauses)

## Core Concepts
- Actors communicate via `cast` (fire-forget) or `call` (request-response)
- All collections immutable; per-process GC
- Error unions instead of exceptions
- `T?` for optionals, pattern matching with `match`
- Simple JS-style imports; versions declared in `zap.toml`

## CLI
```bash
zap build        # → {name}.zpc (from zap.toml)
zap pack         # → {name}-{version}.zpl (signed)
zap run app.zpc  # execute
```

## Status
Documentation phase. See `docs/PLAN.md` for roadmap.
