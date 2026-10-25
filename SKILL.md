---
name: stinner-shaw-method
description: >-
  Review Python code, advise on architecture, and reason about performance
  using a measurement-first methodology derived from Victor Stinner's CPython
  work and Anthony Shaw's teaching: enumerate the objects, write the
  straight-line version, earn abstractions through demonstrated repetition,
  count the interpreter work, and measure with pyperf. Use for Python code
  review, architecture decisions, performance analysis, refactoring guidance,
  and explaining what the interpreter is actually doing.
license: CC-BY-NC-SA-4.0
metadata:
  risk: low
  source: community
  kind: methodology
  category: performance-engineering
  tags: [performance, python, code-review, cpython]
---

# Stinner-Shaw Method: Measurement-First Python Review

## Origin

Derived from the public work of Victor Stinner (CPython core developer,
FASTCALL, pyperf, pyperformance, speed.python.org) and Anthony Shaw
(CPython Internals, Python performance and compiler-internals writing).
Python-language adaptation of the muratori-method skill (TheCasey): the
method structure is Casey Muratori's, translated to Python's cost model.
Attribution only: this file describes a method, not a persona. Style
overlays are the user's choice.

## Method

Follow this order:

1. Enumerate the data as objects. What gets allocated per element or per
   row, how often, and what survives the loop. Python is allocation-first;
   count heap objects before counting bytes.
2. Write the straight-line version first. Plain functions over built-ins
   (list, dict, tuple), no machinery "for later": no ABCs, metaclasses,
   plugin registries, or models with one use.
3. Compress only when a pattern demonstrably repeats. A decorator, base
   class, Protocol, or pydantic model earns its existence from real
   duplication, not anticipated reuse.
4. Count the interpreter work per unit of real work: bytecode dispatches,
   allocations, copies, attribute lookups, boxing, GIL round-trips,
   crossings of the C boundary, and syscalls versus page faults. The C
   boundary is part of the API: each crossing to NumPy, a DB driver, or
   FFI is only cheap when it carries bulk. A page fault is an I/O
   operation the scheduler did for you; it is not free.
5. Measure before and after with pyperf (warmup, multiple processes, quiet
   CPU) or timeit with enough repeats. State the multiplier. An unmeasured
   performance claim is a hypothesis; label it as one.
6. Treat user-visible slowness as a defect, not an optimization
   opportunity.

## Defaults

Hold these positions unless specific evidence overturns them:

- Built-in containers over custom classes; tuple for fixed records;
  __slots__ when many instances are iterated.
- Columnar or bulk data (NumPy arrays, columnar frames) for iterated
  numeric data. A per-element Python loop over what one C call can do is
  a defect.
- numpy.memmap / np.load(mmap_mode=...) for array-like files too large
  for RAM: the array stays on disk and slices like an ndarray. A memmap
  is a view, not a free lunch — operations that materialize the full
  array pay for the whole thing.
- Batch at every boundary: executemany over per-row queries, join over
  +=, one vectorized op over a million scalar ops.
- Comprehensions and local bindings over callback layers in hot loops;
  map/filter with a Python callback is not vectorization.
- No ORM, pydantic validation, decorator stacks, or metaclass machinery
  in a hot path.
- Match the interpreter. Folklore from 3.8 may be false on 3.12+ (PEP 659
  specializing adaptive interpreter); measure on the version you ship.
- Abstraction needs a payer: a concrete, present requirement whose cost
  exceeds the abstraction's cost. "For extensibility" is not a payer.

## The framework position, precisely

The cost center in Python is not polymorphism as such — a method call is
already a dict/protocol lookup. The costs are layers that hide where the
bytes go (DI frameworks, wrapper classes) and allocations hidden behind
convenience (ORM rows, validated models as hot-path DTOs). Duck typing
with plain functions is fine when the variability is real. A Protocol
with exactly one implementation is overhead dressed as foresight.

## The mmap position

Memory-mapping can change the design, not just the speed. Mapping a
file into the address space makes the OS the data path: pages load on
demand via page faults, the page cache is the cache, and write-back is
the flush strategy. Treat the OS as an explicit design layer:

- A file larger than physical memory is workable; only touched pages
  cost RAM. Design the file format for the access pattern: page-aligned
  records, axes you slice along kept contiguous.
- The page cache is shared across processes. Mapping the same file from
  many workers reads one copy of memory with no serialization hop. This
  changes the multiprocessing design: shared page cache beats queues
  for read-only data.
- Mapped bytes are outside the GC's heap. A 40 GB mapping does not
  create 40 GB of Python objects to trace or refcount, and in-place
  views (NumPy over mmap, slicing, re.search over the mapping) avoid
  copying into Python-managed memory entirely.
- Zero-copy reads are the default: reads come from the buffer cache with
  no extra copy into the process. A second pass over the same region is
  RAM-speed; the first pass is still disk-speed. Transparent is not
  fast: demand paging hides disk latency, it does not remove it.
- Access mode is a design decision. Read-only maps are simplest and
  safest. Copy-on-write (ACCESS_COPY / MAP_PRIVATE) gives scratch pages
  without touching the file. Shared writable maps enable cross-process
  state and persistence, but demand msync discipline and synchronization.
- mmap is a default for bulk random access into large structured data
  (logs, binary datasets, array files) — not for small files, and not
  necessarily for sequential scans, where buffered reads may be simpler
  and as fast. Verify on the target OS; empty files, truncation, and
  Windows behavior differ.

## Review checklist

When reviewing code, hunt these in hot loops first:

- Per-element Python loops over NumPy arrays; DataFrame.iterrows or
  row-wise apply
- Hidden allocations and copies: slicing, str +=, df.copy, list building
  inside loops
- Attribute, global, and method lookups inside hot loops; re.compile
  inside a loop
- ORM N+1 queries; schema validation on hot paths
- A C-extension or FFI call per element, boxing every scalar
- Decorator or metaclass machinery in the request path
- Single-run timeit numbers presented as measurements
- Load-then-process pipelines that map-then-process could replace:
  whole-file read() or load() where only slices or a streaming pass are
  touched
- Full materialization of a memmap (np.array(mm), frame conversions)
  that quietly copies the entire file into the heap

## Applicability boundary

Match intensity to the problem. Most Python is glue waiting on I/O —
network, disk, database. If wall time is dominated by waits, interpreter
cost is noise: say so and stop; do not lecture. Scripting and web
handlers are different domains from the interpreter itself, a
data-processing kernel, or a hot library function. A mapping whose
working set is larger than RAM under random access can be slower than a
streaming pass; mmap shifts the design, it does not repeal the disk.

## How to argue

- Show the multiplier, then explain what the interpreter stopped doing:
  bytecodes dispatched, objects allocated, refcounts bumped, boundary
  crossings avoided.
- Explain bottom-up: the eval loop, object layout, the GIL, the C API.
- Use micro-benchmarks correctly (Stinner's position): a speedup visible
  only on a crafted micro-benchmark is not a speedup. Verify against a
  realistic workload or the full suite. Benchmark stability is its own
  problem; an unstable harness produces invented multipliers.
- An interpreter upgrade is often the cheapest multiplier, but
  release-note ranges are measured on their suite, not your workload.
  Measure your own.
- When mmap changes the design, show both data-flow diagrams
  (load-decode-allocate vs map-page-fault), then the multiplier. "mmap
  made it faster" is incomplete; "mmap deleted the load and the copy"
  is an explanation.

## Receipts

These were verified once against the listed sources on 2026-10-25; they
have not been re-verified since:

- Stinner, CPython 2016 Q4: converting re flags to enum.IntFlag slowed
  Python startup from 13.4 ms to 22.8 ms (caught by speed.python.org,
  fixed by not importing re at startup); re.compile() went from 312 ms
  to 376 ms (1.2x slower).
- Stinner rejected the int+int fast-path micro-optimization after two
  years of benchmarking: no effect on non-trivial code, only on crafted
  micro-benchmarks (issue #21955).
- Stinner: Python 3.6 compact dicts are more compact, not faster.
- Faster CPython project: Python 3.11 averaged 1.25x on pyperformance vs
  3.10; workload-dependent 10-60%; startup 10-15% faster; function
  calls 3-7% faster; simple recursion (fibonacci/factorial) 1.7x.
- Muratori's C summation loop ran from 0.006 to 52 adds per cycle
  (~8,000x) on his hardware. His Python follow-up is paywalled: do not
  cite a Python multiplier from it; measure your own.

No mmap multipliers are cited: buffer-cache hit rates are workload- and
hardware-dependent. Measure your own.

Cite these as their numbers and their hardware; do not invent new ones.

## Sources

Grounding references for the human maintainer, not instructions:

- https://vstinner.github.io/contrib-cpython-2016q4.html
- https://vstinner.readthedocs.io/
- https://tonybaloney.github.io/
- Anthony Shaw, CPython Internals (Real Python)
- https://docs.python.org/3/whatsnew/3.11.html (Faster CPython)
- https://www.computerenhance.com/p/python-revisited
- Dave Beazley, "Understanding the Python GIL" (PyCon 2010)
- https://docs.python.org/3/library/mmap.html
- https://numpy.org/doc/stable/reference/generated/numpy.memmap.html
- https://pythonspeed.com/articles/mmap-vs-zarr-hdf5/
