**CSP → Vectorized**
- Purpose: A practical, step‑by‑step recipe to build a vectorized solver for any game modeled in CSP‑Rules. It shows how to map labels and typed variables to boolean tensors, derive binary links (and glinks) from non‑binary constraints, implement ECP/Singles/Subsets/Chains as reductions and set operations, and integrate T&E/DFS — all with slices, reshapes, and broadcasts.

**Preliminaries**
- Goals
  - Keep candidates as boolean tensors; keep counts in small ints; write updates as pure slicing + broadcast; avoid explicit adjacency matrices when structure gives you implicit neighbors.
  - Remain faithful to CSP‑Rules’ semantics: typed variables, link/glink graphs, chain families, and integrated T&E/DFS.
- Canonical Shapes
  - Batch: `B` for puzzles; optional `H` for hypotheses; optional `K` for parallel fronts (chains); optional `P2=2` for parity.
  - Label bank: choose axes per app’s physical indexing. Examples:
    - Sudoku: `C[B, 9, 9, 9]` as `(r, c, d)`.
    - Latin: `C[B, N, N, N]` as `(r, c, d)`.
    - Map: `C[B, |countries|, |colors|]` as `(country, color)`.
    - Slitherlink: stack per‑type banks or add a type axis: e.g., `C[B, T, R, C, V]` with `T∈{H,V,I,P,B}`, `V` enumerating values for that type.
  - G‑labels: add a second bank `G[...]` for grouped nodes; see g‑chains.

**Step 1 — Read The CSP‑Rules Model For Your App**
- Inventory from app sources (and CSP-Docs):
  - Typed variables: e.g., Sudoku `rc/rn/cn/bn`; Latin adds diagonals; Kakuro has run combinations; Slitherlink has vertex/edge degree/loop types.
  - Label encoding and typing: `nrc-to-label`, `csp-var-type`, and app templates (candidates carry structural slots).
  - Link semantics: `labels-linked-by`, `labels-linked`; g‑layer: where `exists-glink`/`csp-glinked` come from.
  - Scheduling: BRT → init‑links/glinks → play families; T&E/DFS optional, using same rules inside child contexts.

**Step 2 — Define Candidate Tensors (Labels → Bits)**
- Choose a consistent axis order and flattening rule for IDs (for logs and joins). Example, Sudoku:
  - `idx = r*81 + c*9 + d0` where `d0∈{0..8}`; recover with `np.unravel_index(idx, (9,9,9))`.
- Initialize `C` from givens via ECP (see Step 5). Keep values separately in `G[B,...]` for display/shortcuts.
- Invariants:
  - For an empty cell: `C[..., r, c, :]` may have many `True`.
  - For a placed cell `(r,c)=digit`: `C[..., r, c, :]` is one‑hot at `d0=digit-1`.

**Step 3 — Binaryize Non‑Binary Constraints (Typed Variables + Links)**
- From CSP‑Docs/Beyond: model each global/non‑binary rule by extra typed variables and/or relational edges:
  - Exactly‑one variables → `csp-linked` between labels sharing the same typed variable.
  - Relational constraints (inequalities, adjacency, distance) → `exists-link` between labels constrained to be mutually exclusive (or to co‑constrain in chains).
- Vector form (no materialized adjacency):
  - Encode each typed variable as a reduction pattern over axes. Example (Sudoku):
    - `rc` per cell → `cell_count = C.sum(axis=-1)`.
    - `rn` row×digit → `row_dn = C.sum(axis=2)` with axes `(row, digit)`.
    - `cn` col×digit → `col_dn = C.sum(axis=1)` with axes `(col, digit)`.
    - `bn` block×digit → reshape to `[..., 3,3, 3,3, 9]`, sum `(-3,-2)` → `(block_row, block_col, digit)`.
  - App‑specific: add diagonal sums (Latin), sector sums and combination compatibility (Kakuro), inequality/adjacency masks (Futoshiki, Map, Slitherlink).

**Step 4 — Implicit Neighbor Masks (Edges Without an Adjacency Matrix)**
- Sudoku example neighbors for `(r,c,d)`:
  - Same row, same `d`: broadcast `C[..., r, :, d]` to mark row peers.
  - Same column, same `d`: broadcast `C[..., :, c, d]`.
  - Same block, same `d`: reshape/tile into `3×3` blocks.
  - Same cell, different `d`: broadcast within `C[..., r, c, :]`.
- Other apps:
  - Futoshiki: precompute pair masks for `<` edges between cells; link digit thresholds appropriately.
  - Map: neighbor pairs per country; same‑color conflicts only.
  - Slitherlink: edges incident to a vertex; loop continuity; treat per‑type neighbor generation.
  - Kakuro: neighbors include glinks between cell digits and sector g‑nodes (combinations/digits).

**Step 5 — ECP (Elementary Constraint Propagation)**
- Single placement `(r,c,d0)` updates, in order:
  - Clear other digits in cell: `C[..., r, c, :] = False`.
  - Remove `d0` from row/col/block peers: slice per structure.
  - Re‑assert the placed one: `C[..., r, c, d0] = True`.
- Batched placements with `P[B,...]` one‑hots:
  - `cell_keep = P | ~P.any(axis=-1, keepdims=True)`
  - For eliminations, compute `elim_mask_row/col/blk` from `P.any` along peer axes.
  - Preserve placements when combining: `C = (C & cell_keep & elim_mask_total) | P`.
  - Never AND placements away; always OR them back last or mask them out from eliminations.

**Step 6 — Basic Strategies As Reductions**
- Singles
  - Naked Single: `cell_count == 1` gives placements `(r,c,d0=argmax)`.
  - Hidden Single in a house: house×digit counts equal to 1 → locate the unique position via `argmax/argwhere`, map back to `(r,c)`.
- Locked Candidates (Pointing/Claiming)
  - Pointing: within a block, if digit `d` appears only in one row stripe (or col stripe), eliminate `d` from that row (or col) outside the block. Use block reshapes + row/col masks.
- Subsets (Pairs/Triples/Quads)
  - Detect k cells ↔ k digits patterns via k‑sized support counts and exact‑cover tests on the house slice; eliminate other digits by boolean set operations.

**Step 7 — Chain Families (Whips/Braids, Typed/G Variants)**
- Frontier masks
  - Keep a frontier `F[B, K, ...]` for active nodes. Add parity `P2=2` for strong/weak alternating layers: `F2[B, K, 2, ...]`.
- Neighbor expansion
  - Derive neighbors on the fly from structure (Step 4) intersected with currently valid candidates `C`. Apply parity gates:
    - Strong links when the typed variable’s count equals 2 (e.g., row/col/block digit count == 2, or cell bivalue for same cell links).
    - Weak links otherwise (conflict edges in the graph).
  - Update: new strong layer from weak neighbors that are exclusive; new weak layer from strong neighbor expansions; mask with availability and not‑already‑visited.
- Targets/eliminations
  - Track a target mask `T[...]`. An elimination occurs when chain semantics prove the target would force a contradiction (e.g., “final cell lacks right candidate”). Detect via typed‑variable zero counts or witnesses within the expansion.

**Step 8 — g‑Chains (Grouped Nodes)**
- Add `G[...]` for g‑candidates and glink adjacency between `C` and `G` (exists‑glink, csp‑glinked in CSP‑Rules).
- Expansion alternates `C ↔ G` steps respecting glink structure, with the same strong/weak logic where applicable.
- Kakuro: sectors’ admissible combinations and digits become g‑nodes; glinks express compatibility.

**Step 9 — T&E and DFS (Hypotheses Axis)**
- Clone along `H`: `C_H[H, B, ...]` with one‑hot `P_H` for hypotheses.
- Propagate to fixpoint per hypothesis using the same kernels (Singles/Subsets/Chains).
- Contradiction signatures: empty `cell_count`, or zero for any typed house×digit count; mark `bad[h]`.
- Forcing comparisons (H=2 on a target):
  - Common eliminations: `(~C_H[0] & ~C_H[1]) & parent_C` → eliminate in parent.
  - Common assertions: same one‑hot in both branches → assert in parent.

**Step 10 — Scheduling**
- Outer loop mirrors CSP‑Rules: BRT → init‑links/glinks equivalents → play families; iterate until quiescence or limits; interleave T&E if enabled.
- Terminate when solved (all typed variables have one candidate) or contradiction.

**Step 11 — Per‑Game Checklist (Does This Mapping Work?)**
- Sudoku (SudoRules)
  - Axes: `(r,c,d)`.
  - Typed reductions: cell, rn, cn, bn.
  - Edges: same row/col/block digit and same cell other digits.
  - g‑layer: optional 2D segments (for g‑whips); map to `G` with rn/cn/bn glinks.
  - Status: All steps above map directly; chains use typed gates from rn/cn/bn counts; T&E/DFS via `H`.
- Latin Squares (+ diagonals)
  - Axes: `(r,c,d)`.
  - Typed reductions: cell, rn, cn; add dn/an for diagonals; include diagonal edges in neighbor derivation.
  - Status: Works with added diagonal reductions and edges.
- Futoshiki
  - Axes: `(r,c,d)`.
  - Typed reductions: cell, rn, cn (like Latin). Inequalities add `exists-link` arcs: for `x<y`, encode relational edges that prune digit orderings; optionally add g‑nodes for ordered segments.
  - Status: Works; implement inequality masks per neighbor pair and digit thresholds.
- Kakuro
  - Axes: `(r,c,d)` for white cells; add `G` bank for sector g‑combinations and g‑digits; glinks assert compatibility.
  - Typed reductions: cell; sector sums handled via `G` constraints; eliminations via `C↔G` propagation.
  - Status: Works with g‑layer; chains can traverse `C↔G` with typed gates.
- Slitherlink
  - Axes: add a type axis or separate banks per edge type; values per type represent permissible states.
  - Typed variables: degree at vertices; loop continuity; map to reductions over incident edges.
  - Edges: conflict edges between incompatible joint assignments; or use g‑nodes for vertex degree constraints.
  - Status: Works; needs a careful type/value layout and degree reductions.
- Map Coloring
  - Axes: `(country, color)`.
  - Typed reductions: one color per country (cell), and optional per‑color totals if needed.
  - Edges: neighbors cannot share a color; derive neighbor masks from the adjacency list.
  - Status: Straightforward; a classic binary CSP fit.
- Hidato / Numbrix
  - Axes: `(r,c,v)` with `v` as the value index.
  - Typed reductions: one value per cell; one cell per value.
  - Edges: `v` and `v+1` must be in adjacent cells; encode adjacency masks over `v` layers.
  - Status: Works; chains traverse along value/adjacency constraints.

**Step 12 — Implementation Patterns (Slices/Views/Reductions)**
- Use reshape to tile blocks or other substructures; keep arrays contiguous when possible (`ascontiguousarray`) to reduce cache misses.
- Prefer boolean logic (`&`, `|`, `~`) over arithmetic; keep counts in `uint8`/`int16`.
- Fuse passes to limit memory traffic; only widen to int for sums.

**Step 13 — Invariants and Validation**
- After each sweep:
  - No empty cells: `cell_count > 0` everywhere.
  - For each typed variable, no zero house×digit counts unless solved.
  - Placements are preserved: `C[P]` must remain True after combined updates.
  - Optionally compute density/metrics akin to CSP‑Rules for sanity.

**Notes**
- Scope warnings: The neighbor derivations shown under Step 4 are Sudoku/Latin‑like. For other apps, derive neighbors from the app’s typed variables or relational definitions (inequalities, adjacency lists, g‑structures).
- Chains need typed gates: compute “strong” availability from the corresponding typed variable counts (e.g., count==2), then restrict parity expansions accordingly.

