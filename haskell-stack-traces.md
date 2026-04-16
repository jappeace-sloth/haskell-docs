# Haskell Stack Traces and Profiling: A Comparison of Approaches

GHC has historically been criticized for poor stack traces.
Unlike languages with explicit call stacks (Java, Python, C with DWARF),
Haskell's lazy evaluation and tail-call optimization mean there is no
conventional call stack to unwind. Several approaches have emerged to
address this, each with different trade-offs.

This report covers four approaches:

1. `HasCallStack` — cooperative source-level annotation
2. Cost-centre profiling (`-prof`) — GHC's built-in profiler
3. Info table maps (`-finfo-table-map`) — lightweight crash diagnostics
4. DWARF debug info (`-g`) — native stack unwinding

## 1. HasCallStack

### How it works

`HasCallStack` is a constraint (from `GHC.Stack`) that you add to function
signatures. GHC threads an implicit `CallStack` parameter through the
annotated functions, recording source locations at each call site.

```haskell
myFunction :: HasCallStack => Int -> Int
myFunction x
  | x < 0     = error "negative input"  -- error uses the CallStack
  | otherwise  = x + 1
```

When `error` is called, the accumulated `CallStack` is included in the
error message, showing the chain of annotated call sites.

### Strengths

- Zero cost where not annotated — only functions with the constraint
  participate.
- Works in any build configuration — no special flags needed.
- Available since GHC 8.0.
- Composable: library authors can expose `HasCallStack` constraints and
  users get call site information through their own code.

### Weaknesses

- Cooperative: you only see frames for functions that have the constraint.
  An unannotated function in the middle of the chain breaks the trace.
- Small runtime cost per annotated call (allocates a `CallStack` value).
- Doesn't help with CPU profiling or performance analysis.
- Must be added pervasively to be useful, which clutters signatures.

### When to use

- For library error messages — adding `HasCallStack` to functions that
  call `error` or `throw` gives users meaningful locations.
- For application-level debugging where you control all the code and can
  annotate the relevant functions.

## 2. Cost-Centre Profiling (-prof -fprof-auto)

### How it works

GHC's built-in profiling system instruments code with "cost centres" —
named points that track time and allocation. When compiled with `-prof`,
the runtime maintains a cost-centre stack (CCS) that records which
cost centres are active.

```bash
# Compile everything with profiling
cabal build --enable-profiling

# Run with cost-centre stack trace on exceptions
./my-program +RTS -xc

# Run with time/allocation profiling
./my-program +RTS -p
# produces my-program.prof

# Heap profiling
./my-program +RTS -h
# produces my-program.hp
```

The `-xc` flag prints cost-centre stacks when exceptions are thrown.
The `-p` flag produces a `.prof` file with time and allocation breakdowns
per cost centre.

### The cost-centre stack model

The CCS is not a real execution stack. It is a logical stack of
annotations that reflects the lexical nesting and dynamic call chain of
cost centres. This means:

- Tail calls don't collapse the stack (unlike native frames).
- The stack reflects which cost centres were entered, not which machine
  code instructions are on the real stack.
- Optimizations like inlining can change what appears in the profile.

### Strengths

- Allocation profiling: the `.prof` file shows exactly where memory is
  being allocated, broken down by cost centre. This is unique to `-prof`.
- Heap profiling (`-h`): produces heap usage over time, essential for
  diagnosing space leaks. Can break down by cost centre, type, closure,
  or retainer.
- `-xc` gives useful exception traces without requiring `HasCallStack`
  annotations.
- Mature and well-documented.

### Weaknesses

- Requires recompiling everything with `-prof` — your code, all
  dependencies, and even boot libraries. Cabal handles this but it
  means a separate build configuration and longer build times.
- Significant runtime overhead: entering and leaving cost centres has
  a cost. Programs run 2-4x slower under profiling.
- Inhibits optimizations: the profiling instrumentation prevents some
  transformations (unboxing, specialization, worker-wrapper) from
  firing. This means the profiled code may behave differently from
  the optimized code.
- Cross-compilation is painful: building all dependencies with `-prof`
  for a cross-compilation target (e.g. Android) requires a separate
  set of profiling libraries for the target architecture.

### When to use

- Diagnosing space leaks — heap profiling is irreplaceable for this.
- Understanding allocation patterns — "which function is allocating
  the most?"
- Getting exception traces in development when `HasCallStack` isn't
  pervasive enough.

## 3. Info Table Maps (-finfo-table-map)

### How it works

Available since GHC 9.2. The `-finfo-table-map` flag embeds source
location information directly in GHC's info tables (the metadata
structures attached to every heap closure). Combined with
`-fdistinct-constructor-tables`, each constructor allocation site
gets its own info table.

```bash
# Compile with info table maps
ghc -finfo-table-map -fdistinct-constructor-tables MyProgram.hs

# Run with backtrace on exception
./my-program +RTS --backtrace-on-exception
```

When an exception occurs, the RTS can walk the stack and look up source
locations from the info tables it finds, producing a trace.

### Strengths

- No recompilation of dependencies required — you only need the flag
  on your own code (though you get less detail for library code).
- Negligible runtime overhead — the source location data sits in the
  binary and is only consulted on exceptions.
- Doesn't inhibit optimizations — the compiled code is the same as
  without the flag.
- Works well with GHC's IPE (Info table Provenance Entries) system
  for richer diagnostics.

### Weaknesses

- Moderate binary size increase (the source location strings are
  embedded in the binary).
- The trace quality depends on what info tables are on the stack at
  the time of the exception — lazy evaluation means the stack may
  not reflect the logical call chain you expect.
- Not useful for CPU profiling — this is purely for crash diagnostics.
- Relatively new, less battle-tested than `-prof`.

### When to use

- Lightweight crash diagnostics in production — get meaningful
  exception traces without the overhead of full profiling.
- When `-prof` is too expensive or impractical (e.g. cross-compilation).
- As a complement to `HasCallStack` for code you don't control.

## 4. DWARF Debug Info (-g)

### How it works

The `-g` flag tells GHC to emit standard DWARF debugging information
in the compiled binary. DWARF is the same debug format used by C, C++,
and Rust compilers. It maps machine code addresses to source locations
and describes the stack frame layout so that external tools can unwind
the native execution stack.

```bash
# Compile with DWARF info
ghc -g MyProgram.hs

# Use perf for CPU profiling
perf record -g ./my-program
perf report

# Use gdb for debugging
gdb ./my-program

# Get stack traces on exceptions
./my-program +RTS --backtrace-on-exception
```

### What is perf?

`perf` is a Linux kernel profiling tool. It uses hardware performance
counters to periodically sample the program's instruction pointer (and
optionally unwind the stack). The result is a statistical profile of
where CPU time is spent.

Without DWARF info, `perf` shows raw addresses or RTS function names:
```
40.0%  0x00007f3a2b001234
35.0%  GarbageCollect
25.0%  stg_ap_p_fast
```

With DWARF, `perf` resolves Haskell source locations:
```
40.0%  MyModule.expensiveFunction  (src/MyModule.hs:42)
35.0%  GarbageCollect
25.0%  OtherModule.helper          (src/OtherModule.hs:17)
```

### Strengths

- No recompilation of dependencies needed — you get full detail for
  your code and at least function-level detail for libraries.
- Negligible runtime overhead — DWARF data is not consulted during
  normal execution. `perf` sampling adds minimal overhead (typically
  under 5%).
- Profiles the real optimized code — unlike `-prof`, optimizations
  are not inhibited. What you measure is what runs in production.
- Works with the entire Linux profiling ecosystem: `perf`, `gdb`,
  `valgrind`, `heaptrack`, FlameGraph, etc.
- Real native stack unwinding — the traces reflect actual execution
  frames, not a logical cost-centre model.

### Weaknesses

- Large binary size increase (2-5x) due to the embedded debug sections.
  The DWARF data can be stripped into a separate `.debug` file to keep
  the deployed binary small.
- No allocation or heap profiling — DWARF tells you about CPU time and
  crash locations, not where memory is being allocated. You still need
  `-prof` for that.
- The native stack may not correspond to what you expect from the
  Haskell source, due to tail calls, lazy evaluation thunks, and GHC's
  continuation-passing compilation. A "call" in Haskell doesn't always
  produce a native stack frame.
- Linux-centric: `perf` is Linux-only. On macOS you'd use Instruments
  or `dtrace` instead. On Windows, support is limited.
- GHC's DWARF support has had bugs historically (unwinding through
  certain RTS frames). It has improved substantially since GHC 9.x.

### When to use

- CPU profiling in realistic conditions — "why is my program slow?"
  without the distortion of `-prof` overhead.
- Crash debugging — getting real stack traces in core dumps or from
  `gdb`.
- Integration with system-level profiling — when you need to see
  Haskell code alongside C/FFI code in the same profile.
- Production monitoring — compile with `-g`, strip debug symbols into
  a separate file, and use them only when investigating issues.

## Comparison Matrix

| Feature                    | HasCallStack     | -prof -xc       | -finfo-table-map | -g (DWARF)       |
|----------------------------|------------------|-----------------|------------------|------------------|
| GHC version                | 8.0+             | ancient         | 9.2+             | 8.0+ (improved 9.x) |
| Recompile dependencies     | no               | yes, all        | no               | no               |
| Runtime overhead           | small (per call) | significant 2-4x| negligible       | negligible       |
| Inhibits optimizations     | no               | yes             | no               | no               |
| Exception stack traces     | annotated only   | yes (-xc)       | yes              | yes              |
| CPU profiling              | no               | yes (time %)    | no               | yes (perf)       |
| Allocation profiling       | no               | yes (.prof)     | no               | no               |
| Heap profiling             | no               | yes (-h)        | no               | no               |
| External tool support      | no               | no              | no               | perf, gdb, etc.  |
| Binary size impact         | none             | separate libs   | moderate         | large (2-5x)     |
| Cross-compilation friendly | yes              | painful         | yes              | yes              |
| Stack model                | logical (call sites) | logical (CCS) | info tables    | native (real)    |

## Recommendations

### For library authors

Add `HasCallStack` to functions that can fail (partial functions,
validators, parsers). This is low-cost and gives users immediate
diagnostic value.

### For application development

Use `-finfo-table-map -fdistinct-constructor-tables` as a default.
The cost is a moderate binary size increase with negligible runtime
overhead, and you get meaningful exception traces without any
source-level annotation.

### For performance investigation

Use DWARF (`-g`) with `perf`. This profiles the real optimized code
without distortion. Generate flame graphs for visual analysis:

```bash
perf record -g ./my-program
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### For space leak diagnosis

Use `-prof` with heap profiling (`-h`). There is no substitute for
GHC's heap profiling when tracking down space leaks. Accept the
recompilation cost and runtime overhead for this specific task.

### For mobile/cross-compilation targets (e.g. Android via hatter)

`-prof` is impractical due to the need to cross-compile all
dependencies with profiling. Use `-finfo-table-map` for crash
diagnostics and DWARF (`-g`) for debug builds. Strip debug
symbols from release APKs.

## Further Reading

- [GHC User's Guide: Profiling](https://downloads.haskell.org/ghc/latest/docs/users_guide/profiling.html)
- [GHC User's Guide: Debugging compiled programs](https://downloads.haskell.org/ghc/latest/docs/users_guide/debug-info.html)
- [Well-Typed: DWARF support in GHC](https://well-typed.com/blog/2020/04/dwarf-1/)
- [GHC User's Guide: Info table maps](https://downloads.haskell.org/ghc/latest/docs/users_guide/rts-opts.html)
- [Brendan Gregg: Flame Graphs](https://www.brendangregg.com/flamegraphs.html)
