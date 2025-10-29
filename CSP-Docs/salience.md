## Contents
- [Purpose](#purpose)
- [TL;DR](#tldr)
- [Original CSP‑Rules Scheduling](#original)
- [What Comes First](#first)
- [Vectorized Scheduling](#vectorized)
- [Non‑Binary Constraints (Beyond Binary)](#nonbinary)
- [Practical Checklist](#checklist)
- [File Pointers](#files)

<a id="purpose"></a>
**Purpose**
- Summarize how CSP‑Rules orders rule families via salience, what fires first, how apps refine the hierarchy, and how to mirror this in a vectorized solver.
- Recap how non‑binary/global constraints are encoded so the whole solver works over binary links.

Reference note: Pointers use file paths and named symbols, not line numbers, to avoid drift across versions.

<a id="tldr"></a>
**TL;DR**
- Phases: initialize → BRT (contradictions, ECP, Singles) → init‑links → play (Subsets, Chains, Uniqueness, Exotic) → optional T&E/DFS.
- Salience and `(technique ?cont …)` facts progressively activate more complex families only when simpler ones no longer apply.
- Vectorized mirror: do fast reductions for BRT and basic subsets, derive implicit neighbor/link masks from structure, then run chain kernels; batch hypotheses/frontiers for T&E/chain breadth.
- Non‑binary constraints are “binaryized” via typed CSP variables (`csp-linked`) and relation edges (`exists-link`). Chains and Singles operate on this binary graph.

<a id="original"></a>
**Original CSP‑Rules Scheduling**
- Phases and gating
  - Solve loop phases: initialize instance → BRT → init‑links → play; optional T&E/DFS reuse the same rule families in child contexts.
  - BRT fires in context 0; after BRT, `init-links` asserts binary links; then `(play)` starts non‑trivial rules per salience.
- BRT order (by salience)
  - 1) Contradiction detection, 2) Elementary Constraint Propagation (ECP), 3) Singles (Naked/Hidden).
- Families and activation
  - Families (Subsets, Chains — bivalue/z/t‑whips/whips/g‑whips/braids — Uniqueness/Deadly, Exotic ORk) are enabled by globals and activated via rules that assert `(technique ?cont <name>)` plus per‑family saliences.
  - Per‑length/per‑variant saliences stage shorter/easier instances before longer/heavier ones (e.g., whips[n] rising with n).
- App‑specific ordering
  - Applications can override/extend ordering with their own `GENERAL/saliences.clp` (e.g., place Uniqueness or Deadly Patterns before/after Singles, raise certain pattern priorities).
- Rationale
  - “Complex techniques are activated progressively, when nothing easier is applicable,” keeping easy instances fast while still enabling deep reasoning when needed.

<a id="first"></a>
**What Comes First**
1) BRT — contradiction detection → ECP → Singles.
2) init‑links/glinks — assert `csp-linked`/`exists-link`/`exists-glink` facts (used by chains and many app rules).
3) play — enable families in an easy→hard progression per salience:
   - Often: local Subsets and Uniqueness/Deadly checks early; then chain families (shorter lengths, typed/g variants later); Exotic ORk sparingly.
4) Optional T&E/DFS around the same rules in child contexts; contradictions feed eliminations back to the parent.

<a id="vectorized"></a>
**Vectorized Scheduling**
- Mirror the same hierarchy:
  - BRT via reductions: cell counts (Naked Singles), row/col/block digit counts (Hidden Singles), plus basic subset detections via axis‑wise sums and boolean masks.
  - Link equivalents: derive implicit neighbor masks from structure (row/col/block, same cell, app relations) instead of materializing adjacency matrices; optionally add grouped/g‑label banks.
  - Chains as batched frontier expansions with parity (strong/weak) and typed gating from availability counts (e.g., “count==2” implies strong links within a typed variable).
  - T&E/DFS as a hypotheses axis `H`: clone states, apply the same kernels to fixpoint, detect contradictions (empty cell or broken typed counts), and aggregate common consequences.
- Iteration and termination: run in sweeps (reductions → deductions → ECP updates) until a fixpoint or configured limits; interleave chain kernels and optional T&E only when simpler passes stall.

<a id="nonbinary"></a>
**Non‑Binary Constraints (Beyond Binary)**
- Core idea
  - Introduce extra, typed CSP variables whose domains enumerate mutually exclusive choices; connect labels via `is-csp-variable-for-label`; assert pairwise `csp-linked` for labels sharing a typed variable.
  - Add `exists-link` for pure relational constraints that aren’t “exactly‑one” (inequality, adjacency, distance). Optionally add g‑labels and glinks for grouped constraints (e.g., Kakuro sectors).
- Effect
  - All constraints become a uniform binary link graph (`csp-linked`/`exists-link` plus optional `exists-glink`) on which Singles, Subsets, and Chains operate unchanged across domains.
- Examples (by app)
  - Sudoku: `rc/rn/cn/bn` typed variables binaryize AllDifferent.
  - Futoshiki: inequalities add relation edges in addition to AllDifferent csp‑links.
  - Kakuro: sector sums via g‑labels; glinks connect digits to sector combinations.
  - Map: country color choice as a CSP variable; neighbor constraints as non‑csp links between same‑color labels across neighbors.
  - Hidato/Numbrix: per‑cell and per‑value CSP variables; adjacency/distance as relation links.
  - Slitherlink: typed variable families and precomputed physical links promoted to `csp-linked`/`exists-link`.

<a id="checklist"></a>
**Practical Checklist**
- Original (CLIPS)
  - Ensure BRT saliences: contradictions → ECP → Singles.
  - Run `init-links` before asserting `(play)`; compute/print density if needed.
  - Gate families with globals; activate via `(technique ?cont …)`; order by increasing complexity/length with saliences.
  - Let apps adjust ordering (e.g., Uniqueness/Deadly before/after Singles, special modules, ORk activation caps).
- Vectorized
  - Implement BRT/subsets with reductions and boolean masks first.
  - Compute structural neighbor masks on the fly (row/col/block, same cell; app relations) and optional g‑label connectivity.
  - Implement chain kernels with parity and typed gating; increment length caps progressively.
  - Add an `H` axis for T&E; detect contradictions via empty cells or broken typed counts; apply common consequences to the parent.

<a id="files"></a>
**File Pointers**
- Generic phases, salience, and play: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp`, `.../saliences.clp`, `.../init-links.clp`, `.../play.clp`
- Technique gating pattern (example typed whips): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-*/**`
- App saliences (Sudoku example): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/saliences.clp`
- Docs overview of phases and families: `CSP-Docs/Model.md`, `CSP-Docs/Trigger.md`, `CSP-Docs/Overview.md`
- Non‑binary encoding details and app examples: `CSP-Docs/Beyond.md`
- Vectorized approach: `CSP-Vectorized-Proposal/Vectorized.md`, `CSP-Vectorized-Proposal/CSP-to-Vectorized.md`

