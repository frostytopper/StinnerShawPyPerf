# StinnerShawPyPerf

> Notes from the human: this is still in testing phase. Currently I use
> Perplexity for access to models, and that fundamentally changes behavior.
> I will report back on using this with some example files in a bit using
> some of the pro models available to me currently. I have until 12/29/26
> until my sub runs out, and I'll start testing the frontier models in T3
> code with this skill later.

Measurement-first Python code review and performance methodology derived
from Victor Stinner's CPython work and Anthony Shaw's teaching: enumerate
the objects, write the straight-line version, earn abstractions through
demonstrated repetition, count the interpreter work, and measure with
pyperf. mmap is treated as a design lever, not an optimization. Host- and
agent-agnostic; loadable by any system that supports skills.

See [SKILL.md](SKILL.md) for the method; see [NOTES.md](NOTES.md) for
provenance, design decisions, and proposed evaluation cases.

## Related

- [TheCasey](https://github.com/frostytopper/TheCasey) (private) — the Muratori-lineage parent methodology this skill was adapted from
- [collaborative-workbench](https://github.com/frostytopper/collaborative-workbench) (private) — where the method gets exercised: the ceiling-first protocol in [`projects/mmos`](https://github.com/frostytopper/collaborative-workbench/tree/main/projects/mmos), [`log-analyzer-perf-ladder`](https://github.com/frostytopper/collaborative-workbench/tree/main/projects/log-analyzer-perf-ladder), [`saas-perf-lab`](https://github.com/frostytopper/collaborative-workbench/tree/main/projects/saas-perf-lab), the free-threaded CPython harness in [`projects/ft-bench`](https://github.com/frostytopper/collaborative-workbench/tree/main/projects/ft-bench), and the companion design skill [`skills/architecture_explained`](https://github.com/frostytopper/collaborative-workbench/blob/main/skills/architecture_explained/SKILL.md) (its bound-type annotation pass feeds this one: the annotation is the prediction, the profiler is the verdict)

## License

CC-BY-NC-SA-4.0 — see [LICENSE](LICENSE).
