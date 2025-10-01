**State Model In CSP‑Rules**

State = facts in working memory + globals. Facts capture the live problem (candidates, variables, links, contexts, chains). Globals capture configuration toggles, counters, and small caches used by rules and output.

**Facts (Working Memory)**
- Core candidates
  - `candidate` — per placement, with `status` `cand` or `c-value`, plus `context` and `flag` (T&E). Generic template: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:49`. Apps extend with structural slots (e.g., Sudoku adds `number/row/column/block` in `SudoRules-V20.1/GENERAL/templates.clp:22`).
  - `g-candidate` — grouped labels for g‑chains; has `type` and optional `csp-var`: `CSP-Rules-Generic/GENERAL/templates.clp:83` (generic), app variants (e.g., Sudoku `SudoRules-V20.1/GENERAL/templates.clp:41`).

- Structural model
  - `csp-variable` — one per typed “slot” (e.g., Sudoku `rc/rn/cn/bn`): created per app; used to detect contradictions: “no label left for a csp-variable”.
  - Label↔variable relations:
    - `is-csp-variable-for-label` and `is-csp-variable-for-glabel`: `CSP-Rules-Generic/GENERAL/templates.clp:128`, `:144`.
    - Typed variants for typed chains: `is-typed-csp-variable-for-label/glabel`: `CSP-Rules-Generic/GENERAL/templates.clp:152`, `:160`.

- Links (binary graph edges over labels)
  - Effective edges asserted at init per context:
    - `csp-linked` (same typed variable ⇒ mutual exclusion) and `exists-link` (any app‑level adjacency): computed in `CSP-Rules-Generic/GENERAL/init-links.clp:49` (rules at :78, :102). Apps can override (e.g., Latin adds diagonals: `LatinRules-V2.1/GENERAL/init-links.clp:74`).
  - Grouped edges (when g‑labels are active):
    - `csp-glinked`, `exists-glink` from labels to `g-candidate`s; see Sudoku’s `SudoRules-V20.1/GENERAL/init-glinks.clp:61` and generic glinks summary `CSP-Rules-Generic/GENERAL/init-glinks.clp:213`.
  - Some apps seed “physical” edges first, then init‑links derives effective edges (e.g., Map `physical-link`: `MapRules-V2.1/GENERAL/init-links.clp:24`; Slither `physical-csp-link/physical-link`: `SlitherRules-V2.1/GENERAL/init-links.clp:33`).

- Contexts (for T&E/DFS)
  - `context (name, parent, depth, generating-cand…)`: `CSP-Rules-Generic/GENERAL/templates.clp:266`.
  - Orchestration helpers: `technique` (phase marker), `phase`, `phase-productive-in-context`, `clean-and-retract` (created/consumed by T&E rules in `CSP-Rules-Generic/T&E+DFS/*`).

- Chains and patterns (derived reasoning state)
  - `chain`, `typed-chain`, `csp-chain`, `chain2r`, `ORk-chain`, `ORk-relation`: `CSP-Rules-Generic/GENERAL/templates.clp:292`, `:312`, `:330`, `:356`, `:392`, `:433`.
  - These are asserted/transient while rules fire; they carry the sequence (llcs/rlcs/csp-vars) and are context‑scoped via `context` slot.

- Focus/utility
  - `candidate-in-focus` for narrowing search/printing (optional): `CSP-Rules-Generic/GENERAL/templates.clp:176`.
  - App‑specific helpers (e.g., Kakuro `sector-with-gcombs`), not core to state.

Key points about facts
- Context‑scoped: Most facts either have a `context` slot (candidates, g‑candidates, chains) or carry context in their first field (e.g., `csp-linked ?cont a b`).
- Monotonicity: candidates only move from `cand` → `c-value` or are retracted; no re‑addition in the same context.
- Links are “physical” adjacency, independent of eliminations; rules guard on remaining candidates when traversing them.

**Globals (Toggles, Counters, Caches)**
- Universal per‑instance counters (reset by `init-universal-globals`)
  - `?*nb-csp-variables*`, `?*nb-csp-variables-solved*`, `?*nb-candidates*`, `?*nb-g-candidates*`.
  - Link metrics: `?*csp-links-count*`, `?*links-count*`, `?*csp-glinks-count*`, `?*glinks-count*`; density `%`: `?*density*` computed as `density(nb-cands, nb-links)`: `CSP-Rules-Generic/UTIL/utils.clp:190`.
  - Time: `?*init-instance-time*`, `?*solve-instance-time*`, `?*total-instance-time*`.
  - Context/search: `?*context-counter*`, `?*max-depth*`, `?*DFS-max-depth*`, `?*solution-found*`.
  - Lists for output: `?*label-links*`, `?*label-glabel-glinks*`, `?*label-in-glabel*`, `?*glabel-in-glabel*`.
  Pointers: `CSP-Rules-Generic/GENERAL/globals.clp:68`–`:210`; init at `:135`–`:178`.

- Feature toggles (resolution theory)
  - Families: `?*Subsets*`, `?*Bivalue-Chains*`, `?*z-Chains*`, `?*t-Whips*`, `?*Whips*`, `?*Braids*`, typed/g variants; ORk options; max lengths. See `CSP-Rules-Generic/GENERAL/globals.clp:399`–`:579`, `:735`–`:775`.
  - Behaviour switches: `?*blocked-Subsets*`, `?*blocked-chains*`, `?*unblocked-behaviour*`: `:378`–`:387`.
  - Chain implementation mode: `?*chain-rules-optimisation-type*` = SPEED/MEMORY: `:360`.
  - T&E/DFS toggles: `?*TE1*`, `?*TE2*`, `?*TE3*`, `?*DFS*`, `?*Forcing-TE*`: `:753`–`:775`.
  - Printing: `?*print-actions*`, `?*print-solution*`, `?*print-RS-after-Singles*`, many per‑technique flags; generic at `:1801+`, app‑specific at `SudoRules-V20.1/GENERAL/globals.clp:653+`.

- Small caches (label pair sets)
  - `?*links*`, `?*glinks*` hold pairs `(label,label)` or `(label,glabel)` for quick `linked`/`glinked` membership tests: created via `add-link`/`add-glink` in `CSP-Rules-Generic/GENERAL/generic-background.clp:230`, `:273`.
  - Maintained for context 0 only (init‑links calls `add-link` only when `cont = 0`). Child contexts recompute link facts but don’t update these caches.
  - Some apps maintain extra caches (e.g., Futoshiki `?*labels-ineq-links*` for inequalities: `FutoRules-V2.1/GENERAL/background.clp:318`).

Key points about globals
- Globals persist across `reset` (engine configured with `(set-reset-globals FALSE)`): `CSP-Rules-Generic/GENERAL/globals.clp:26`–`:36`. Each run calls `init-universal-globals` to reset per‑instance counters and caches; feature toggles remain as you set them.
- Apps can extend globals and override `init-specific-globals` (e.g., Sudoku prints, Tridagon flags): `SudoRules-V20.1/GENERAL/globals.clp:113`, `:286+`.

**Lifecycle & State Transitions**
- Init
  - `init-universal-globals` resets counters/caches; app creates `csp-variable`s and asserts `is-csp-variable-for-label` for all labels.
  - The instance asserts candidates (`cand`), givens as `c-value`, then `context 0` and `technique 0 BRT` are set. See generic solve model: `CSP-Rules-Generic/GENERAL/solve.clp:93`–`:157`.
- BRT (Singles + ECP)
  - Eliminates candidates and asserts values; updates `?*nb-candidates*`, `?*nb-csp-variables-solved*` in context 0.
- init‑links and play
  - `init-links` asserts `csp-linked`/`exists-link` (and glinks) and updates link counters/caches (context 0 only). Density updated in `play`: `CSP-Rules-Generic/GENERAL/play.clp:42`.
  - `play` asserts `(play)` and later families (chains/subsets/uniqueness) fire per salience.
- T&E/DFS (optional)
  - New `context` facts are created; parent’s `candidate`/`c-value` facts are duplicated into child; link facts recomputed; contradiction detection uses `csp-variable` coverage tests; child facts are retracted on cleanup. See `CSP-Rules-Generic/T&E+DFS/T&E1.clp:23`, `:86`, `:56`.
- End / Results
  - `record-results` prints final counts/density if configured: `SudoRules-V20.1/GENERAL/record-results.clp:216`.

**Mutation Semantics (What Changes, What Doesn’t)**
- Candidates
  - Eliminations retract `candidate` facts; assertions set `status` to `c-value`. In context 0, counters are adjusted; in child contexts (T&E), `flag 0` c‑values don’t trigger ECP.
- Links
  - Effective link facts (`csp-linked`/`exists-link`) are asserted at init and remain valid regardless of subsequent eliminations; chain rules check both adjacency and current `candidate` presence.
  - Caches `?*links*`/`?*glinks*` record “physical” adjacency for context 0 and are not decremented when candidates are removed (not needed for correctness).
- Globals
  - Counters and density only intended for context 0; child contexts use their own facts to detect contradictions/solutions.

**Scoping & Performance Notes**
- Context slot: All rules are context‑safe — they only read/write facts with a consistent `context`.
- `linked` implementation: apps may override generic `linked` to compute adjacencies from structure (e.g., Sudoku `linked` delegates to `labels-linked`): `SudoRules-V20.1/GENERAL/background.clp:484`–`:492`. This avoids heavy dependence on caches.
- SPEED vs MEMORY chains: globals select which chain implementation directory is loaded and thus affect transient chain facts and performance.

**Observability & Output**
- `?*print-*` flags control banners, step traces, intermediate resolution states (after Singles), final RS, and stats files; generic printing in `solve.clp/play.clp`, app printing in `GENERAL/record-results.clp`.
- `watch rules/facts` can be toggled in the CLIPS REPL for debugging heavy state transitions.

**Where To Hook New State**
- Add structural slots to `candidate`/`g-candidate` in your app’s `GENERAL/templates.clp`.
- Add new typed CSP variables and their constructors in `GENERAL/background.clp`; emit `is-csp-variable-for-label` and override `labels-linked(-by)` to expose adjacencies.
- If you need group membership, define `g-candidate`s and `is-csp-variable-for-glabel` and seed glinks in `GENERAL/init-glinks.clp`.
- Register any extra caches or counters in your app’s `GENERAL/globals.clp` and reset them in `init-specific-globals`.

**See Also**
- Model: objects and API surface — [Model](Model.md)
- Overview: load order and modules — [Overview](Overview.md)
- Pattern taxonomy: where facts are matched — [Trigger](Trigger.md)
- Output notation: reading state in traces — [Notation](Notation.md)
- T&E / DFS: how state is copied/cleaned in contexts — [T&E](T&E.md)
- Non‑binary → binary modeling — [Beyond](Beyond.md)
