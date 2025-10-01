**Vectorized Foundations**
- State tensors
  - `grid G`: `np.uint8[9,9]` with `0` for empty, `1..9` for placed values. Batched form: `np.uint8[B,9,9]` for B puzzles/contexts. See similar vocabulary in `Docs/Model.md:1` and single/ECP references in `Docs/Overview.md:1`.
  - `candidates C`: `np.bool_[9,9,9]` where `C[r,c,n]` is “digit `n+1` is allowed at `(r,c)`”. Batched: `np.bool_[B,9,9,9]`.
  - Contracts: at any time, `G[r,c] == 0` ⇒ `C[r,c,:]` may have many `True`; `G[r,c] == k` ⇒ `C[r,c,:]` is a one‑hot at `n=k-1`.
- Fast reductions (broadcasting + axis‑wise sums)
  - Per‑cell counts: `cell_count = C.sum(axis=-1)` → `np.int16[...,9,9]`. Naked Single where `cell_count == 1`.
  - Row‑digit counts: `row_dn = C.sum(axis=2)` → shape `[...,9,9]` indexed as `(row, digit)`. Hidden Single in row where `row_dn == 1` gives the column via `np.argmax` on columns.
  - Col‑digit counts: `col_dn = C.sum(axis=1)` → shape `[...,9,9]` indexed as `(col, digit)`.
  - Block‑digit counts: reshape to `C.reshape(...,3,3,3,3,9)` as `[..., BR, BC, br, bc, n]`; then `blk_dn = C_blk.sum(axis=(-3,-2))` → shape `[...,3,3,9]` for `(block, digit)`. Hidden Single in block where `blk_dn == 1` gives `(br,bc)` by `np.argwhere` inside the 3×3 slice.
- Propagation (ECP) as pure slicing
  - When asserting `(r,c) ← n` (0‑based `n` for digit `n+1`):
    - Zero out other digits in cell: `C[r,c,:] = False; C[r,c,n] = True`.
    - Remove `n` from row peers: `C[r,:,n] = False` except `c`.
    - Remove `n` from column peers: `C[:,c,n] = False` except `r`.
    - Remove `n` from block peers: `br,bc = r//3,c//3; C[br*3:(br+1)*3, bc*3:(bc+1)*3, n] = False` except `(r,c)`.
  - Batched placement mask `P`: `np.bool_[B,9,9,9]` with one‑hots per batch; propagate without loops via broadcasted logical updates, e.g. set‑wise:
    - Keep only placed digit in each placed cell: `cell_keep = P | ~P.any(axis=-1, keepdims=True)`; apply `C &= cell_keep`.
    - Row/col/block eliminations computed from `P.any` over the appropriate axes and masked back into `C` with broadcast; no `matmul`/`einsum` required.
- Basic strategies as reductions
  - Naked Single: `cell_count == 1` → placements P.
  - Hidden Single (row/col/block): counts `== 1` on the corresponding reductions; locate the lone `(r,c)` (or `(br,bc)` → `(r,c)`).
  - Locked candidates (pointing/claiming): use block row/col projections; e.g., for a fixed digit `n`, “only in a block’s row stripe” is `C_blk[..., BR,BC, :, :, n].any(axis=block_col_or_row)` combined with row/col availability masks, followed by broadcast eliminations.
  - Subsets (pairs/triples/quads): detect `k` positions by `axis` sums equal to `k` and exact‑cover masks with boolean comparisons; apply eliminations by clearing other digits in the covered cells via broadcast logic.

**Batch‑Parallel Chain Reasoning**
- From graphs to tensors
  - Conflict links in `Docs/Graphs.md:1` (candidate–candidate edges for same cell/row/column/block or app constraints) map to structure‑aware boolean expansions. Instead of materializing a 729×729 adjacency, derive neighbors on the fly from `(r,c,n)` structure with axis reductions.
  - Treat each chain “frontier” as a batch of parallel contexts: add a frontier dimension `K` and keep `F: np.bool_[B,K,9,9,9]` for current active nodes in each parallel chain instance.
- Neighbor expansion without `einsum`
  - Row neighbors for a frontier `F`: `row_hit = F.any(axis=3, keepdims=True)` over columns; broadcast back across columns to mark all `(r,*,n)` in that row for each context. Symmetric for columns: `col_hit = F.any(axis=2, keepdims=True)`.
  - Block neighbors: reshape `F` to `[...,3,3,3,3,9]`, reduce `any` over the inner `br,bc` axes, then broadcast back into the 3×3 tiles.
  - Same‑cell neighbors (different digits): `cell_hit = F.any(axis=-1, keepdims=True)` and broadcast across digits.
  - Combined neighbor mask: `N = (row_hit | col_hit | blk_hit | cell_hit)` intersected with currently valid candidates `C` and pruned by `~F` (no self‑loops). All operations use boolean broadcasting and axis‑wise reductions.
- Parity and link types
  - Chain families alternate link “strengths” (e.g., strong/weak). Model parity by adding a length‑parity axis `P2=2`: `F2[B,K,2,9,9,9]` where `F2[...,0]` holds “strong” layer and `F2[...,1]` the “weak” layer at the current depth.
  - Update step toggles parity: new strong layer from weak neighbors that are exclusive, new weak layer from strong neighbors per allowed link type; both computed with the same reductions but masked by parity‑appropriate constraints (e.g., uniqueness in a house gives strong links when count==2).
- Targets and eliminations (vector form)
  - For eliminations typical of whips/braids, track per‑context “target” candidate(s) as a mask `T[B,K,9,9,9]`. Detect contradictions in the expansion (e.g., a house losing all candidates unless T is False) by axis‑wise zero sums under hypothetical toggles; mark eliminations where the target would force contradiction.
  - Stop conditions per context: reach of fixed max length, fixpoint (`F` layers no longer grow), or a contradiction signature. All can be computed with boolean comparisons and `any/sum` tests per context in parallel.
- Grouped/typed variants
  - `g‑chains` in `Docs/Graphs.md:1` extend the node set with g‑candidates. Represent them as additional axes or concat a separate boolean bank for g‑labels with glink expansions implemented by the same block‑style reshapes and reductions.
  - Typed chains restrict neighbor formation to typed CSP variables (rn/cn/bn); mask neighbor expansions by typed availability computed from `C`’s row/column/block projections.

**Batched Hypothesis Search**
- Contexts as a batch dimension
  - T&E/DFS in `Docs/T&E.md:1` duplicate facts into child contexts and run the same rules. Mirror this by cloning tensors along a hypotheses axis `H` to get `C_H: np.bool_[H,B,9,9,9]` (or just `np.bool_[H,9,9,9]` if unbatched input).
  - Build `H` by selecting hypotheses (e.g., all bivalue cells or a curated subset), making `P_H` one‑hot placements, then asserting them into copies: apply the ECP propagation step to each `C_H[h]` via broadcasted updates.
- Propagate to fixpoint per hypothesis
  - On each `C_H[h]`, run the same vectorized BRT/subsets/chain kernels until fixpoint: repeated rounds of reductions → deductions → ECP updates. With an `H` axis, every kernel is the same as basics, just with one more leading dimension.
  - Contradiction detection (no graph needed):
    - Empty cell: `cell_count == 0` anywhere in `C_H[h]`.
    - Broken typed variables (see `Docs/Beyond.md:1`): row/col/block digit counts zero after propagation, i.e., `row_dn == 0` or `col_dn == 0` or `blk_dn == 0` for some `(house, digit)`.
  - Mark branches with contradictions via a boolean `bad[h]` and gather indices for parent‑level eliminations: if hypothesis `(r,c,n)` led to contradiction, eliminate candidate `n` at `(r,c)` in the parent by clearing `C[:,r,c,n]` (or `C[r,c,n]` without an outer `B`).
- Forcing T&E and comparisons
  - For bivalue forcing, use `H=2` per target: clone two branches with complementary one‑hot `P_H`, propagate, then compute:
    - Common eliminations: candidates cleared in both branches (`~C_H[0] & ~C_H[1]`) → clear in parent.
    - Common assertions: new one‑hots asserted in both branches (cell_count==1 with the same `argmax`) → assert in parent.
  - All comparison logic is boolean set algebra across the `H` axis; no `matmul`/bitsets required.
- Scheduling and interoperability
  - Mirror `Docs/Overview.md:1` and `Docs/State.md:1`: run BRT → init‑links‑equivalents (our reductions) → chain kernels; between outer parent steps, interleave batched T&E where enabled. Terminate early if a full sweep makes no changes.
  - Performance tips: keep everything boolean until writing to `G`; use `np.uint8` for counts; fuse passes to minimize memory traffic; exploit `np.ascontiguousarray` and views for the 3×3 block reshapes.

**Notes**
- Philosophy: Adopt the binary‑constraint graph from `Docs/Graphs.md:1` and the “non‑binary via extra variables” pattern from `Docs/Beyond.md:1`, but realize them as structure‑aware boolean tensor operations. Broadcasting and axis‑wise sums give adjacency and propagation without explicit adjacency matrices, `einsum`, or bitsets.
- Portability: The same layout (`[B,9,9,9]` with optional `[K]`, `[H]`, and parity axes) lifts directly to JAX (`jnp`) or PyTorch (`torch.bool`, `torch.uint8`), with `vmap`/`vectorized_map` over batch axes for free parallelism.
