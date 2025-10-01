## Contents
- [State Model](#state)
- [Facts (Working Memory)](#facts)
- [Globals](#globals)
- [Lifecycle](#lifecycle)
- [Mutation Semantics](#mutation)
- [Scoping & Performance](#scoping)
- [Observability & Output](#observe)
- [Hook New State](#hook)
- [See Also](#see)

<a id="state"></a>
**State Model In CSP‑Rules**

State = facts in working memory + globals. Facts capture the live problem (candidates, variables, links, contexts, chains). Globals capture configuration toggles, counters, and small caches used by rules and output.

Reference note: Pointers use file paths and named symbols (no line numbers) to avoid drift across versions.

<a id="facts"></a>
**Facts (Working Memory)**
- Core candidates
  - `candidate` — per placement, with `status` `cand` or `c-value`, plus `context` and `flag` (T&E). Generic template: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`. Apps extend with structural slots (e.g., Sudoku adds `number/row/column/block` in `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp`).
  - `g-candidate` — grouped labels for g‑chains; has `type` and optional `csp-var`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp` (generic), app variants (e.g., Sudoku `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp`).

- Structural model
  - `csp-variable` — one per typed “slot” (e.g., Sudoku `rc/rn/cn/bn`): created per app; used to detect contradictions: “no label left for a csp-variable”.
  - Label↔variable relations:
    - `is-csp-variable-for-label` and `is-csp-variable-for-glabel`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`.
    - Typed variants for typed chains: `is-typed-csp-variable-for-label/glabel`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`.

- Links (binary graph edges over labels)
  - Effective edges asserted at init per context:
    - `csp-linked` (same typed variable ⇒ mutual exclusion) and `exists-link` (any app‑level adjacency): computed in `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`. Apps can override (e.g., Latin adds diagonals: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1/GENERAL/init-links.clp`).
  - Grouped edges (when g‑labels are active):
    - `csp-glinked`, `exists-glink` from labels to `g-candidate`s; see Sudoku’s `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp` and generic glinks summary `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-glinks.clp`.
  - Some apps seed “physical” edges first, then init‑links derives effective edges (e.g., Map `physical-link`: `CSP-Rules/CSP-Rules-V2.1/MapRules-V2.1/GENERAL/init-links.clp`; Slither `physical-csp-link/physical-link`: `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/GENERAL/init-links.clp`).

- Contexts (for T&E/DFS)
  - `context (name, parent, depth, generating-cand…)`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`.
  - Orchestration helpers: `technique` (phase marker), `phase`, `phase-productive-in-context`, `clean-and-retract` (created/consumed by T&E rules in `CSP-Rules-Generic/T&E+DFS/*`).

- Chains and patterns (derived reasoning state)
  - `chain`, `typed-chain`, `csp-chain`, `chain2r`, `ORk-chain`, `ORk-relation`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`.
  - These are asserted/transient while rules fire; they carry the sequence (llcs/rlcs/csp-vars) and are context‑scoped via `context` slot.

- Focus/utility
  - `candidate-in-focus` for narrowing search/printing (optional): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`.
  - App‑specific helpers (e.g., Kakuro `sector-with-gcombs`), not core to state.

Key points about facts
- Context‑scoped: Most facts either have a `context` slot (candidates, g‑candidates, chains) or carry context in their first field (e.g., `csp-linked ?cont a b`).
- Monotonicity: candidates only move from `cand` → `c-value` or are retracted; no re‑addition in the same context.
- Links are “physical” adjacency, independent of eliminations; rules guard on remaining candidates when traversing them.

<a id="globals"></a>
**Globals (Toggles, Counters, Caches)**
- Universal per‑instance counters (reset by `init-universal-globals`)
  - `?*nb-csp-variables*`, `?*nb-csp-variables-solved*`, `?*nb-candidates*`, `?*nb-g-candidates*`.
  - Link metrics: `?*csp-links-count*`, `?*links-count*`, `?*csp-glinks-count*`, `?*glinks-count*`; density `%`: `?*density*` computed as `density(nb-cands, nb-links)`; see `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/UTIL/utils.clp`.
  - Time: `?*init-instance-time*`, `?*solve-instance-time*`, `?*total-instance-time*`.
  - Context/search: `?*context-counter*`, `?*max-depth*`, `?*DFS-max-depth*`, `?*solution-found*`.
  - Lists for output: `?*label-links*`, `?*label-glabel-glinks*`, `?*label-in-glabel*`, `?*glabel-in-glabel*`.
  Pointers: see `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp` for definitions and initialisation.

- Feature toggles (resolution theory)
  - Families: `?*Subsets*`, `?*Bivalue-Chains*`, `?*z-Chains*`, `?*t-Whips*`, `?*Whips*`, `?*Braids*`, typed/g variants; ORk options; max lengths. See `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp`.
  - Behaviour switches: `?*blocked-Subsets*`, `?*blocked-chains*`, `?*unblocked-behaviour*` (in generic `globals.clp`).
  - Chain implementation mode: `?*chain-rules-optimisation-type*` = SPEED/MEMORY (in generic `globals.clp`).
  - T&E/DFS toggles: `?*TE1*`, `?*TE2*`, `?*TE3*`, `?*DFS*`, `?*Forcing-TE*` (see generic `globals.clp`).
  - Printing: `?*print-actions*`, `?*print-solution*`, `?*print-RS-after-Singles*`, many per‑technique flags; see generic `globals.clp` and app‑specific `SudoRules-V20.1/GENERAL/globals.clp`.

- Small caches (label pair sets)
  - `?*links*`, `?*glinks*` hold pairs `(label,label)` or `(label,glabel)` for quick `linked`/`glinked` membership tests: created via `add-link`/`add-glink` in `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp`.
  - Maintained for context 0 only (init‑links calls `add-link` only when `cont = 0`). Child contexts recompute link facts but don’t update these caches.
  - Some apps maintain extra caches (e.g., Futoshiki `?*labels-ineq-links*` for inequalities: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/background.clp`).

Key points about globals
- Globals persist across `reset` (engine configured with `(set-reset-globals FALSE)`): see `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp`. Each run calls `init-universal-globals` to reset per‑instance counters and caches; feature toggles remain as you set them.
- Apps can extend globals and override `init-specific-globals` (e.g., Sudoku prints, Tridagon flags): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp`.

<a id="lifecycle"></a>
**Lifecycle & State Transitions**
- Init
  - `init-universal-globals` resets counters/caches; app creates `csp-variable`s and asserts `is-csp-variable-for-label` for all labels.
  - The instance asserts candidates (`cand`), givens as `c-value`, then `context 0` and `technique 0 BRT` are set. See generic solve model: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp`.
- BRT (Singles + ECP)
  - Eliminates candidates and asserts values; updates `?*nb-candidates*`, `?*nb-csp-variables-solved*` in context 0.
- init‑links and play
  - `init-links` asserts `csp-linked`/`exists-link` (and glinks) and updates link counters/caches (context 0 only). Density updated in `play`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/play.clp`.
  - `play` asserts `(play)` and later families (chains/subsets/uniqueness) fire per salience.
- T&E/DFS (optional)
  - New `context` facts are created; parent’s `candidate`/`c-value` facts are duplicated into child; link facts recomputed; contradiction detection uses `csp-variable` coverage tests; child facts are retracted on cleanup. See `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/T&E+DFS/T&E1.clp`.
- End / Results
  - `record-results` prints final counts/density if configured: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/record-results.clp`.

<a id="mutation"></a>
**Mutation Semantics (What Changes, What Doesn’t)**
- Candidates
  - Eliminations retract `candidate` facts; assertions set `status` to `c-value`. In context 0, counters are adjusted; in child contexts (T&E), `flag 0` c‑values don’t trigger ECP.
- Links
  - Effective link facts (`csp-linked`/`exists-link`) are asserted at init and remain valid regardless of subsequent eliminations; chain rules check both adjacency and current `candidate` presence.
  - Caches `?*links*`/`?*glinks*` record “physical” adjacency for context 0 and are not decremented when candidates are removed (not needed for correctness).
- Globals
  - Counters and density only intended for context 0; child contexts use their own facts to detect contradictions/solutions.

<a id="scoping"></a>
**Scoping & Performance Notes**
- Context slot: All rules are context‑safe — they only read/write facts with a consistent `context`.
- `linked` implementation: apps may override generic `linked` to compute adjacencies from structure (e.g., Sudoku `linked` delegates to `labels-linked`): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`. This avoids heavy dependence on caches.
- SPEED vs MEMORY chains: globals select which chain implementation directory is loaded and thus affect transient chain facts and performance.

<a id="observe"></a>
**Observability & Output**
- `?*print-*` flags control banners, step traces, intermediate resolution states (after Singles), final RS, and stats files; generic printing in `solve.clp/play.clp`, app printing in `GENERAL/record-results.clp`.
- `watch rules/facts` can be toggled in the CLIPS REPL for debugging heavy state transitions.

<a id="hook"></a>
**Where To Hook New State**
- Add structural slots to `candidate`/`g-candidate` in your app’s `GENERAL/templates.clp`.
- Add new typed CSP variables and their constructors in `GENERAL/background.clp`; emit `is-csp-variable-for-label` and override `labels-linked(-by)` to expose adjacencies.
- If you need group membership, define `g-candidate`s and `is-csp-variable-for-glabel` and seed glinks in `GENERAL/init-glinks.clp`.
- Register any extra caches or counters in your app’s `GENERAL/globals.clp` and reset them in `init-specific-globals`.

<a id="see"></a>
**See Also**
- Model: objects and API surface — [Model](Model.md)
- Overview: load order and modules — [Overview](Overview.md)
- Pattern taxonomy: where facts are matched — [Trigger](Trigger.md)
- Output notation: reading state in traces — [Notation](Notation.md)
- T&E / DFS: how state is copied/cleaned in contexts — [T&E](T&E.md)
- Non‑binary → binary modeling — [Beyond](Beyond.md)
