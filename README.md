# CSP-Analysis

This workspace collects analysis and guides for understanding CSP‑Rules and its applications, with a focus on source‑first reading (no execution required).

## Docs — Start Here

- [Overview](Docs/Overview.md) — Architecture at a glance, directory map, how to load and use modules.
- [Notation](Docs/Notation.md#quickstart) — How to read output quickly (chains, whips, braids) and real excerpts.
- [Model](Docs/Model.md) — Mental model and public API: variables, labels, links, chains.
- [Graphs](Docs/Graphs.md) — What the solver’s graphs are (nodes/edges), how they’re built and used; code‑anchored.
- [State](Docs/State.md) — What “state” means: facts (candidates, csp‑vars, links) + globals (toggles, counters, caches).
- [Trigger (Pattern Taxonomy)](Docs/Trigger.md) — Scope → Trigger → Action mental model and a taxonomy of pattern families.
- [Beyond (Non‑binary → Binary)](Docs/Beyond.md) — How extra typed variables binaryize global constraints in each app.
- [T&E / DFS](Docs/T&E.md) — Trial & Error and Depth‑First Search: contexts, contradiction detection, and integration with rules.

## Repos In Workspace

- `CSP-Rules/` — Local copy of CSP‑Rules V2.1 (core + apps + docs + CLIPS binaries).
- `CSP-Rules-Examples/` — Example puzzles and large‑scale studies (per app).

Shortcuts (browse code):

- Generic output helpers: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-output.clp`
- Print symbols: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/parameters.clp`
- Generic init‑links: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`
- Sudoku NRC output: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/nrc-output.clp`

If you’re new, begin with Overview, then Notation (to decode trace lines fast), followed by Model and State. Use Trigger as your taxonomy reference, consult Beyond when you need app‑specific binaryization details, and use T&E for branching mechanics. This order optimizes for quick comprehension of solver output before diving into internals.
