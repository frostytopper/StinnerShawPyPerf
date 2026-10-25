# Stinner-Shaw method skill: notes

## Provenance

This skill is a Python-language adaptation of TheCasey (frostytopper),
the muratori-method skill, license CC-BY-NC-SA-4.0 unchanged. The method
structure — data enumeration -> straight-line implementation -> earned
compression -> machine-work accounting -> measurement — is Casey
Muratori's; the Python translation is grounded in the public work of
Victor Stinner (measurement discipline, receipts) and Anthony Shaw
(internals teaching). As with the original, attribution only: no
persona, voice, or style performance is included, and the file contains
no tool, connector, host, or environment references.

## Design decisions

- **Method translated, not rewritten:** the six-step order and the
  applicability boundary carry over in structure. "Machine work" becomes
  interpreter work: bytecode dispatches, allocations, boxing, boundary
  crossings, syscalls vs page faults.
- **The C boundary replaces the cache as the API surface:** Muratori's
  "the cache is part of the API" becomes "the C boundary is part of the
  API" — every NumPy/DB/FFI crossing pays tax and only earns it back
  when it carries bulk.
- **Defaults translated:** SoA becomes columnar/bulk data; switch-over-
  vtable becomes batched C calls over per-element Python callbacks;
  arenas become "don't allocate at all" (built-ins, __slots__, memmaps).
- **Measurement hardened per Stinner:** pyperf with warmup, multiple
  processes, and a quiet CPU is the default; single-run timeit numbers
  are anecdotes, not measurements. Micro-benchmarks that only win on
  crafted snippets are rejected (his int+int case).
- **Interpreter-version disclaimer added:** folklore from 3.8 may be
  false on 3.12+ (PEP 659 specializing adaptive interpreter); measure on
  the version you ship.
- **mmap position added:** mapping a file is a data-path decision (OS
  paging vs explicit load) made at design time, not a micro-optimization.
  The section states when it restructures the design — shared page cache
  across processes, off-GC-heap bulk data, in-place parsing — and when it
  does not (sequential scans, small files, random access with a poor
  working set). No mmap multipliers are cited: hit rates are workload-
  and hardware-dependent.
- **Receipts verified once, on 2026-10-25:** Stinner 2016 Q4 (startup
  13.4->22.8 ms from the re/enum import; re.compile() 312->376 ms),
  the rejected int+int fast path, compact dicts not faster; Python 3.11
  Faster CPython numbers from the official What's New. Muratori's
  Python follow-up is paywalled, so no Python multiplier is claimed
  from it.
- **Sources extended with mmap references:** Python mmap docs, NumPy
  memmap docs, and a buffer-cache analysis (pythonspeed.com).

## Suggested editing priorities

1. Test against the evaluation cases below before iterating on wording.

## Proposed evaluation cases

These are future tests, not completed evaluations. Run the same inputs
against a baseline agent with and without the skill and compare the
trace.

| Case | Example task | Observable success |
| --- | --- | --- |
| NumPy scalar loop | Review element-wise array loop | Names per-call boxing cost; proposes one batched/vectorized op with a pyperf plan |
| Micro-benchmark dogma | timeit shows 2x on a crafted snippet | Rejects as evidence; demands realistic workload or suite benchmark |
| I/O-bound boundary | Review web handler waiting on DB | States the interpreter is not the constraint; no lecture |
| Premature machinery | pydantic model with one use in a hot loop | Flags payer-less abstraction; proposes plain dict/tuple |
| Measurement discipline | Before/after shown as single runs | Demands pyperf, repeats, stable CPU |
| Interpreter folklore | "X is faster than Y" folklore from 3.8 | Requires re-measuring on the target version; cites PEP 659 |
| mmap design lever | Pipeline reads a 30 GB binary dataset, slices ~1 GB | Proposes memmap or mmap with layout matched to the access pattern; names the deleted load/copy steps; flags page-fault cost |
| mmap dogma check | Small config file read once per run | Says mapping is not the constraint; buffered read is simpler |

## Grounding references

These informed the adaptation; they are not dependencies of the skill.

- https://vstinner.github.io/contrib-cpython-2016q4.html
- https://vstinner.readthedocs.io/
- https://tonybaloney.github.io/
- https://docs.python.org/3/whatsnew/3.11.html (Faster CPython)
- https://docs.python.org/3/library/mmap.html
- https://numpy.org/doc/stable/reference/generated/numpy.memmap.html
- https://pythonspeed.com/articles/mmap-vs-zarr-hdf5/
- https://www.computerenhance.com/p/python-revisited
- Dave Beazley, "Understanding the Python GIL" (PyCon 2010)
