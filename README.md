# StinnerShawPyPerf

##Notes from the human - This is still in testing phase. Currently
I use Perplexity for access to models, and that fundamentally changes
behavior. I will report back on using this with some example files in a bit
using some of the pro models available to me currently. I have until 12/29/26
until my sub runs out and I'll start testing the frontier models in T3 code
with this skill later.##

Measurement-first Python code review and performance methodology derived
from Victor Stinner's CPython work and Anthony Shaw's teaching: enumerate
the objects, write the straight-line version, earn abstractions through
demonstrated repetition, count the interpreter work, and measure with
pyperf. mmap is treated as a design lever, not an optimization. Host- and
agent-agnostic; loadable by any system that supports skills. See SKILL.md;
see NOTES.md for provenance, design decisions, and proposed evaluation
cases.
