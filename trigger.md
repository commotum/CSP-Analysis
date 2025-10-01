**Pattern Triggers In CSP‑Rules**

This document explains how every rule fits a simple Scope → Trigger → Action mental model, and organizes the main pattern families (a taxonomy) you’ll find in CSP‑Rules.

**Mental Model (Scope → Trigger → Action)**
- Scope (what we look at)
  - A bounded set of facts: candidates/csp‑variables inside a unit (row/col/block), a sector/run, or along link‑reachable candidates (chains). For chain families, scope is the local subgraph built from `csp-linked`/`exists-link` (and optionally `exists-glink`).
- Trigger (condition)
  - A recognizable arrangement in scope: e.g., “two cells share only the same two numbers” (Naked Pair), or “a chain of length n linking target to a contradiction” (whip[n]). Implemented as the left‑hand side (LHS) of a CLIPS `defrule` matching templates (candidates, links, typed relations) and tests.
- Action (consequence)
  - The rule’s right‑hand side (RHS). Typically:
    - Eliminate candidate(s) (retract a `candidate` with status `cand`).
    - Assert a value (turn a `candidate` into `c-value`).
    - Occasionally set orchestration/diagnostics facts (e.g., `(technique ?cont ...)`), or update counters/prints.

Execution pipeline and gating
- Phases (by salience):
  1) BRT (Singles + ECP), 2) init‑links (compute `csp-linked`/`exists-link`), 3) play (chains, subsets, uniqueness, exotic), optionally 4) T&E/DFS. Pointers: `CSP-Rules-Generic/GENERAL/solve.clp`, `.../saliences.clp:448`, `.../play.clp`.
- Families are enabled or blocked via globals (e.g., `?*Whips*`, `?*Subsets*`, `?*blocked-chains*`). Pointers: `CSP-Rules-Generic/GENERAL/globals.clp:378`–`:579`.

**Taxonomy (Pattern Families)**

1) Core Propagation
- BRT — Basic Resolution Theory
  - Scope: current context’s candidates per csp‑variable.
  - Trigger: local necessity/sufficiency conditions.
  - Action: eliminate or assert values.
  - Subtypes:
    - ECP (Elementary Constraint Propagation): remove candidates conflicting with givens or immediate constraints. Files: `CSP-Rules-Generic/GENERAL/ECP.clp`.
    - Singles:
      - Naked Single (unique candidate left for a cell) and Hidden Single (only place for a number in a unit). Generic scaffolding: `.../Single.clp`; app specifics, e.g., Sudoku `SudoRules-V20.1/GENERAL/NS.clp`, `HS.clp`.
- Contradiction Detection (structural)
  - Scope: a csp‑variable and its mapped labels.
  - Trigger: no candidate left (used in T&E/DFS and to validate solutions in DFS).
  - Action: eliminate a hypothesis in parent context (T&E), or assert solution found (DFS). Files: `T&E+DFS/T&E1.clp:86`, `DFS.clp:150`.

2) Subset‑Style (Set‑Based) Patterns (Application Layer)
- Scope: labels inside a unit/run/sector or app‑specific grouping.
- Trigger: combinatorial profiles (pair/triple/quad; fish; wing patterns).
- Action: eliminate affected candidates elsewhere in the unit/run/sector.
- Sudoku examples (directory `SudoRules-V20.1/SUBSETS`):
  - Naked/Hidden Pairs/Triplets/Quads: `N2-`, `N3-`, `N4-`, `H2-`, `H3-`, `H4-`.
  - Fish (row/column interactions): `SH2-x-wing`, `SH3-swordfish`, `SH4-jellyfish`; finned variants `FSH2/3/4-...`.
  - Specials: `SpN4-`, `SpH4-`, `SpSH4-`.
  - Wings/XY‑family (if enabled) appear among subset files.
- Kakuro sectors (sum constraints) use g‑labels:
  - Scope: a run’s sector; Trigger: admissible sum combinations; Action: prune digits/cells inconsistent with combinations. See `KakuRules-V2.1/GENERAL/glabels.clp` and `solve.clp` around sector construction.

3) Chain Families (Graph‑Walk Patterns)
- Scope: the link graph over labels (and optionally glabels) built at init‑links: `csp-linked`/`exists-link`/`exists-glink`.
- Trigger: an alternating chain meeting the family’s constraints (typed/untyped, left/right linking semantics) between a target and a contradiction/support.
- Action: eliminate the target (or assert a consequence) when the chain’s semantics guarantee it.
- Major families (generic dirs under `CSP-Rules-Generic/CHAIN-RULES-*`):
  - Bivalue‑Chains
  - z‑Chains
  - t‑Whips
  - Whips
  - g‑Whips (use glinks/g‑candidates)
  - Braids
  - Typed variants (restrict to chosen `csp-var-type`s), and speed/memory variants.
- Typed chains (2D, per app typing):
  - Restrict chain steps to specific `csp-var-type` (e.g., Sudoku 2D chains). Requires `is-typed-csp-variable-for-label` and app `csp-var-type` mapping. Templates at `CSP-Rules-Generic/GENERAL/templates.clp:152`.
- Bi‑Chains (Contradiction chains):
  - Bi‑whips/Bi‑braids derive eliminations from contradiction ends; see bi‑TE and contradiction‑chain scaffolding in templates.

4) ORk / Forcing Chains (Exotic)
- Scope: ordinary link graph plus a named ORk relation (e.g., Tridagon); rule instances thread before/after the OR fragment.
- Trigger: an allowed ORk relation on k candidates combined with pre/post chain segments meeting typed/untyped semantics.
- Action: targeted eliminations or assertions (Forcing vs Contrad variants).
- Generic dirs: `CSP-Rules-Generic/CHAIN-RULES-EXOTIC/*` (e.g., `OR2-Whips`, partial OR3 g‑whips). ORk templates: `CSP-Rules-Generic/GENERAL/templates.clp:433`.
- App‑level OR relations:
  - Sudoku Tridagons and anti‑tridagons (modules under `SudoRules-V20.1/MODULES/TRID-*`); flags in `SudoRules-V20.1/GENERAL/globals.clp:286+`.

5) Uniqueness / Deadly Patterns (Application Layer)
- Scope: configurations that would lead to multiple solutions (deadly rectangles/patterns, BUG states).
- Trigger: a recognizable deadly structure (UR1…UR4, BUG, DP6/8/9/12 in subfolders).
- Action: eliminate guardian candidates or assert values to preserve uniqueness.
- Sudoku dirs: `SudoRules-V20.1/UNIQUENESS/*` (e.g., `UR1.clp`, `BUG.clp`, `Deadly-Patterns/*`).

6) Templates / Advanced / Goodies (Application Layer)
- Templates as patterns (rare, domain‑specific) — see `SudoRules-V20.1/TEMPLATES` if present.
- Advanced helpers: e.g., “eleven replacement”, “mark k‑digit patterns”, disable/re‑enable families (`SudoRules-V20.1/ADVANCED/*.clp`).
- Goodies/printing/shuffling utilities (`SudoRules-V20.1/GOODIES/*`).

7) Search Augmentations (Control, not pattern families)
- T&E(n) and DFS: create child contexts and run the same pattern families there; conclusions feed back to the parent (eliminations) or detect solution. Files: `CSP-Rules-Generic/T&E+DFS/*`.

**Condition–Action Anatomy Of A Rule (Typical)**
- LHS (Scope + Trigger):
  - Bind a `context`; match a named `technique` phase (for ordering); collect candidate labels and their typed variables; walk links/glinks or sector membership; ensure exclusivity with `not` and `forall` conditions.
- RHS (Action):
  - Eliminate `(retract (candidate ...))` or assert a value `(modify cand (status c-value))`; update counters for context 0; print/record technique if configured.

**Prerequisites For Each Family**
- Subsets: app templates expose unit membership (row/col/block/run); scope is a set of labels in the unit.
- Chains: require `init-links` to have asserted link facts; for g‑chains also need `g-candidate`s and `init-glinks`.
- Typed chains: app defines `csp-var-type` and asserts typed mappings (`is-typed-csp-variable-for-label`).
- ORk chains: app asserts OR‑relation facts and modules; generic ORk templates available.
- Uniqueness/Deadly: app templates/derived facts to detect deadly structures.

**Putting It Together (How To Think While Reading Rules)**
- Identify the family (taxonomy above) and confirm its prerequisites are loaded (globals enabled; modules present).
- Read the rule’s Scope in LHS: what unit/run/graph area is being matched? where do candidates/glinks/typed vars come from?
- Identify the Trigger: the exact shape (counts, link alternations, OR sections, subset cardinalities).
- Map the Action: which labels are eliminated or asserted, which counters/prints fire — verify they’re limited to the rule’s `context`.

**Code Map (Where They Live)**
- Generic chain families: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-*/*` (COMMON, SPEED, MEMORY, EXOTIC).
- Generic core (BRT, links, salience, templates): `CSP-Rules-Generic/GENERAL/*`.
- Sudoku application (canonical example): `SudoRules-V20.1/GENERAL`, `/SUBSETS`, `/UNIQUENESS`, `/EXOTIC`, `/MODULES`.
- Other applications (LatinSquares/Futoshiki/Kakuro/Hidato/Numbrix/Slitherlink/Map): mirror the same structure per domain.

**Are There More?**
- The major families above cover the solver’s public resolution theory. Additional domain‑specific helpers (e.g., sector combinatorics in Kakuro) and exotic modules (Oddagon/Tridagon‑based) fit under Subset/Chain/ORk umbrellas. Search augmentations (T&E/DFS) are control strategies, not resolution patterns, but they reuse the same Scope→Trigger→Action rules inside child contexts.

**Quick Reference**
- BRT/ECP — per csp‑variable | immediate conflicts from givens/structure | eliminate candidates; assert forced values | CSP-Rules-Generic/GENERAL/ECP.clp:1, .../Single.clp:1
- Singles (NS/HS) — unit (row/col/block) | only place for a number / only value in a cell | set `c-value` | SudoRules-V20.1/GENERAL/NS.clp:1, HS.clp:1
- Subsets (Naked/Hidden) — unit | k cells ↔ k digits (or hidden variant) | eliminate peers’ digits in unit | SudoRules-V20.1/SUBSETS/N2-*.clp, H2-*.clp
- Fish (X‑/Sword‑/Jellyfish) — rows×cols pattern | aligned cover sets | eliminate candidates in fins/columns/rows | SudoRules-V20.1/SUBSETS/SH2-x-wing.clp:1, SH3-swordfish.clp:1, SH4-jellyfish.clp:1
- Chains (Whips/t‑Whips) — link graph (`csp-linked`/`exists-link`) | alternating chain length n from target to contradiction | eliminate target | CSP-Rules-Generic/CHAIN-RULES-SPEED/WHIPS/*
- g‑Whips — link+glink graph | alternating chain using g‑candidates | eliminate target | CSP-Rules-Generic/CHAIN-RULES-*/G-WHIPS/*, init-glinks SudoRules-V20.1/GENERAL/init-glinks.clp:22
- Braids — link graph | braid support variant reaching contradiction | eliminate target | CSP-Rules-Generic/CHAIN-RULES-*/BRAIDS/*
- Typed Chains (2D) — link graph with `csp-var-type` | chain constrained to chosen types | eliminate target | typed templates CSP-Rules-Generic/GENERAL/templates.clp:152, app `csp-var-type`
- ORk/Forcing Chains — link graph + ORk relation | valid OR fragment + pre/post chains | eliminate or assert (forcing) | CSP-Rules-Generic/CHAIN-RULES-EXOTIC/*, SudoRules-V20.1/MODULES/TRID-*
- Uniqueness/Deadly — unit/block rectangles/patterns | deadly rectangle/BUG/DP structure | eliminate guardians or assert | SudoRules-V20.1/UNIQUENESS/UR*.clp, BUG.clp, Deadly-Patterns/*
- Kakuro Sectors — run (horiz/verti) | sum combinations/compatible digits | prune cells/digits | KakuRules-V2.1/GENERAL/glabels.clp:60, solve.clp:360
- Futoshiki Inequalities — row/col | inequality arcs + AllDifferent | prune by order constraints | FutoRules-V2.1/GENERAL/background.clp:318, :339
- Map Neighbourhood — countries | same‑colour on adjacent countries | eliminate same‑colour neighbours | MapRules-V2.1/GENERAL/init-links.clp:38
- Slitherlink Degree/Loop — edges/vertices | degree/loop constraints per type | prune edge states | SlitherRules-V2.1/GENERAL/S.clp:61, init-links.clp:60
