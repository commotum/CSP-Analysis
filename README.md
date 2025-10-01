# CSP-Analysis

This workspace collects analysis and guides for understanding CSP‑Rules and its applications, with a focus on source‑first reading (no execution required).

## Docs — Start Here

- [Overview](Docs/Overview.md) — Architecture at a glance, directory map, how to load and use modules.
- [Model](Docs/Model.md) — Mental model and public API: variables, labels, links, chains.
- [State](Docs/State.md) — What “state” means: facts (candidates, csp‑vars, links) + globals (toggles, counters, caches).
- [Beyond (Non‑binary → Binary)](Docs/Beyond.md) — How extra typed variables binaryize global constraints in each app.
- [Trigger (Pattern Taxonomy)](Docs/Trigger.md) — Scope → Trigger → Action mental model and a taxonomy of pattern families.
- [Notation](Docs/Notation.md) — What the console output means (chains, whips, braids), per‑game notation, and real excerpts.
- [T&E / DFS](Docs/T&E.md) — Trial & Error and Depth‑First Search: contexts, contradiction detection, and integration with rules.
- [Itinerary](Docs/Itinerary.md) — Research/checklist placeholder (optional roadmap).

## Repos In Workspace

- `CSP-Rules/` — Local copy of CSP‑Rules V2.1 (core + apps + docs + CLIPS binaries).
- `CSP-Rules-Examples/` — Example puzzles and large‑scale studies (per app).

Shortcuts (browse code):

- Generic output helpers: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-output.clp`
- Print symbols: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/parameters.clp`
- Generic init‑links: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`
- Sudoku NRC output: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/nrc-output.clp`

If you’re new, begin with Overview, then Model and State, and use Trigger + Notation as your quick references while reading traces. Consult Beyond when you need to understand how an app encodes its global constraints, and T&E for branching mechanics.
