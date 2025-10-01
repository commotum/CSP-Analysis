## Contents
- [Where Output Lives](#where)
- [Quick Start](#quickstart)
- [Quick Excerpts](#quickexcerpts)
- [Core Symbols](#symbols)
- [Labels and Cells](#labels)
- [Chain Line Anatomy](#chains)
- [Other Prints](#other)
- [Helpers Index](#helpers)
- [Per‑Game Notation](#pergame)
- [End‑to‑End Excerpts](#excerpts)
- [See Also](#see)

<a id="where"></a>
**Notation And Output In CSP‑Rules**

This guide explains how CSP‑Rules formats its console output when verbosity is high, and where the helper functions live. It also decodes the notation for pattern‑based reasoning lines (chains, whips, braids), which follow a single stream of reasoning per pattern.

**Where Output Is Implemented**
- Generic helpers
  - Chain and candidate printing: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-output.clp:1`
  - Print symbols and tokens (signs, braces, separators): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/parameters.clp:39`
  - Banners and timing: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp:1` (`print-start-banner`, `print-banner`)
- Application overrides
  - Sudoku NRC format (labels, pairs, csp‑var names): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/nrc-output.clp:1`
  - Latin Squares pretty grid output: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1/GENERAL/pretty-print.clp:86`
  - Sudoku partial solution printing, summaries: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/print-partial-sol.clp:34`, `.../record-results.clp:1`

<a id="quickstart"></a>
**Quick Start (Read Chains In 30 Seconds)**
- Look for the header: technique + length, e.g., `whip[5]:` or `biv-chain[3]:`.
- Read each cell as a CSP‑variable context with two values: `{val1 val2}`; links are shown with ` - `.
- For non‑reversible families (whips/braids), the final cell prints `{val .}` to mark the missing right candidate.
- The implication shows the conclusion: ` ==> not <label>` (elimination) or ` ==> assert <label>` (forcing).
- One printed line = one linear derivation (single stream of reasoning); ORk families show side branches but still yield a single conclusion.

<a id="quickexcerpts"></a>
**Quick Excerpts**
- Sudoku — Metcalf‑B7B
  - `whip[5]: r6n3{c1 c5} - r4n3{c5 c8} - c8n5{r4 r3} - r2n5{c7 c6} - r2n3{c6 .} ==> r5c1 ≠ 3`
- Map Colouring — Tatham30
  - `biv-chain[3]: France {Blue Red} - Germany {Red Green} - Czechia {Green .} ==> France ≠ Red`
See more in End‑to‑End Excerpts below.

<a id="symbols"></a>
**Core Symbols (Generic Defaults)**
- Signs and separators (from `.../GENERAL/parameters.clp`)
  - Implication: `?*implication-sign*` → ` ==> ` `CSP-Rules-Generic/GENERAL/parameters.clp:41`
  - Link in chains: `?*link-symbol*` → ` - ` `CSP-Rules-Generic/GENERAL/parameters.clp:43`
  - Start/End cell: `?*starting-cell-symbol*` → `{`, `?*ending-cell-symbol*` → `}` `CSP-Rules-Generic/GENERAL/parameters.clp:45`
  - Inside cell separator: `?*separation-sign-in-cell*` → space `CSP-Rules-Generic/GENERAL/parameters.clp:47`
  - Dot placeholder (final missing rlc): `?*dot-in-cell*` → `.` `CSP-Rules-Generic/GENERAL/parameters.clp:48`
  - Non‑equal and equal: `?*non-equal-sign*` → `≠` `CSP-Rules-Generic/GENERAL/parameters.clp:40`, `?*equal-sign*` → `=` `:39`
- Label tokens (short names)
  - Number/Row/Column prefixes: `?*number-symbol*` = `n`, `?*row-symbol*` = `r`, `?*column-symbol*` = `c` `CSP-Rules-Generic/GENERAL/parameters.clp:69`
  - Numeral prefix (often empty): `?*numeral-symbol*` (empty by default) `:68`

Applications adjust these as needed (e.g., Slitherlink sets additional loop link symbols; Kakuro prints run/sector descriptors).

<a id="labels"></a>
**Label And Cell Printing (Sudoku example)**
- Label formatting (one candidate):
  - Sudoku overrides `print-label` to print number+row+column short names: `5r2c9` `SudoRules-V20.1/GENERAL/nrc-output.clp:24`.
- Bivalue cell (two candidates in one CSP‑variable):
  - Printed as `Block {val1 val2}` depending on CSP type (rc, rn, cn, bn). The value pair appears inside `{}` separated by a space: `nrc-output.clp:45` and `:69`.
- Final cell in non‑reversible chains (whips/braids):
  - Printed with a dot for the missing last right‑linking candidate: `{val .}` `SudoRules-V20.1/GENERAL/nrc-output.clp:88`.

<a id="chains"></a>
**Chain/Whip/Braid Line Anatomy (Single Stream)**
- Generic (reversible) chains (bivalue, g‑bivalue, oddagon):
  - `biv-chain[n]: <cell1> - <cell2> - ... ==> not <label>` `CSP-Rules-Generic/GENERAL/generic-output.clp:113`
- Generic (non‑reversible) chains (z‑chain, t‑whip, whip, braid, g‑whip):
  - `whip[n]: <cell1> - ... - <final-cell{val .}> ==> not <label>` `CSP-Rules-Generic/GENERAL/generic-output.clp:299`, `:319`
- Typed chains:
  - `biv-chain-<type>[n]: ... ==> not <label>` (type = `rc`, `rn`, `cn`, etc.) `CSP-Rules-Generic/GENERAL/generic-output.clp:227`
- ORk forcing/contrad variants:
  - Multi‑branch sections are printed beneath a header with `||` to show the k subchains, yet the output is still one pattern instance producing one conclusion: `CSP-Rules-Generic/GENERAL/generic-output.clp:519`.

Interpretation
- Header: technique name and length, e.g., `whip[5]:`.
- Steps: each `<cell>` is a CSP‑variable context and a pair `{val1 val2}` (or an app‑specific pair); steps are joined by ` - `.
- Final cell (non‑reversible families): uses `{val .}` to indicate the last missing candidate.
- Conclusion: ` ==> not <label>` for eliminations or ` ==> assert <label>` for assertions in forcing variants.
- “Single stream of reasoning”: every printed line encodes one linear derivation with no OR branching inside the main stream; ORk families print side subpaths, but the pattern instance still yields one final elimination or assertion.

<a id="other"></a>
**Other Notable Prints**
- Resolution state snapshots
  - After Singles (optional): banner + “Resolution state after Singles” with counts and density. Triggered in `play.clp` if `?*print-RS-after-Singles*` is TRUE.
  - Pretty grid dumps (Latin Squares): `LatinRules-V2.1/GENERAL/pretty-print.clp:86` via `pretty-print-current-resolution-state`.
  - Sudoku partial grid prints: `SudoRules-V20.1/GENERAL/print-partial-sol.clp:34`.
- Banners and timing
  - Start banner and end banner include app/version, engine, rating‑type, and machine descriptor; times printed as `init-time`, `solve-time`, `total-time` when `?*print-time*`/`?*print-actions*` are enabled. See `CSP-Rules-Generic/GENERAL/solve.clp:20` (`print-start-banner`, `print-banner`).
- Level/phase traces
  - When `?*print-levels*` is TRUE, rules announce entry into higher‑level families (e.g., “Entering_level_tridagon[12]”). These are family‑specific debug prints.

<a id="helpers"></a>
**Helper Functions Index (Generic)**
- Print label/cell pairs: `print-label`, `print-bivalue-cell`, `print-final-cell` `CSP-Rules-Generic/GENERAL/generic-output.clp:30`, `:63`, `:280`
- Print chain lines:
  - Reversible: `print-bivalue-chain`, `print-g-bivalue-chain`, `print-oddagon` `generic-output.clp:135`, `:145`, `:171`
  - Non‑reversible: `print-z-chain`, `print-t-whip`, `print-whip`, `print-braid`, `print-gwhip`, `print-gbraid` `generic-output.clp:317`–`:365`
  - Typed variants: `print-typed-bivalue-chain`, etc. `generic-output.clp:236`–`:259`
- Assertions/Eliminations: `print-asserted-candidate`, `print-deleted-candidate` `generic-output.clp:210`
- ORk printing: `print-ORk-forcing-...` families `generic-output.clp:519`+

**Helper Functions Index (Applications)**
- Sudoku: `SudoRules-V20.1/GENERAL/nrc-output.clp:1` (overrides print‑label, bivalue/final cells, candidate assertions/elims), `SudoRules-V20.1/GENERAL/print-partial-sol.clp:34`, `SudoRules-V20.1/GENERAL/record-results.clp:1`
- Latin Squares: `LatinRules-V2.1/GENERAL/pretty-print.clp:86`, `.../nrc-output.clp:1`
- Others (Futoshiki, Kakuro, Hidato, Slitherlink, Map) have their own nrc/pretty/print modules tuned to the domain; search for `nrc-output.clp`, `pretty-print`, or `print-partial-sol` under each app’s `GENERAL`.

**Tips For Reading Traces**
- Look for the technique header (`whip[n]`, `biv-chain[n]`, `g-whip[n]`, `braid[n]`, ...), then scan the `{ }` pairs along the chain to see which values are linked; the final `==>` tells you what was eliminated or asserted.
- For typed chains you’ll see `-<type>[n]` in the header; the cells reflect the chosen type (e.g., `rn` shows a row and a number paired with a list of columns).
- ORk lines show subpaths under the `||` bars; the last line still ends with a single `==>` conclusion.

<a id="pergame"></a>
**Per‑Game Notation**

Sudoku (SudoRules)
- Label format: `<number><row><column>` e.g., `5r2c9` via `print-label` `SudoRules-V20.1/GENERAL/nrc-output.clp:24`.
- CSP types in cells:
  - `rc`: `r2c9 {5 7}`; `rn`: `r2 n5 {c1 c9}`; `cn`: `c9 n5 {r2 r8}`; `bn`: `b3 n5 {r2c7 r2c8}`.
- Eliminations and assertions: `r2c9 ≠ 5` and `r2c9 = 5` via `print-deleted-candidate`/`print-asserted-candidate` `SudoRules-V20.1/GENERAL/nrc-output.clp:111`, `:120` (uses `r/c/n` tokens from generic parameters).
- Chain example: `whip[4]: r2c9 {5 7} - r2n5 {c1 c9} - c1n5 {r2 r7} - r7c1 {5 .} ==> not 5r2c9`.
Examples
- Chain: `whip[4]: r2c9 {5 7} - r2n5 {c1 c9} - c1n5 {r2 r7} - r7c1 {5 .} ==> r2c9 ≠ 5`
  Plain English: Assuming r2c9 is 5 forces a sequence that ends in a contradiction, so r2c9 cannot be 5.
- Elimination: `r3c7 ≠ 4`
  Plain English: 4 at r3c7 conflicts with current constraints, so it’s removed.

Latin Squares (LatinRules)
- Label format: `<number><row><column>`; pretty grid printer for current state: `LatinRules-V2.1/GENERAL/pretty-print.clp:86`.
- Additional diagonals (if pandiagonal enabled): `dn`/`an` appear in cells with diagonal identifiers: e.g., `D3 n5 {r2c3 r4c1}` `LatinRules-V2.1/GENERAL/nrc-output.clp:34`.
- Eliminations/assertions: `r2c3 ≠ 7`, `r2c3 = 7` `LatinRules-V2.1/GENERAL/nrc-output.clp:109`, `:120`.
- Chains use same generic chain printers; cells show `rc/rn/cn` and optionally `dn/an` contexts.
Examples
- Chain (with diagonal): `z-chain[3]: r2c1 {5 7} - dn D3 n5 {r2c1 r4c3} - r4c3 {5 .} ==> r2c1 ≠ 5`
  Plain English: If r2c1 were 5, diagonal D3 would force an impossible continuation at r4c3, so r2c1 ≠ 5.
- Elimination: `r5c6 ≠ 3`
  Plain English: 3 at r5c6 contradicts unit/diagonal constraints, so it’s eliminated.

Futoshiki
- Label format: `<number><row><column>` (same as Latin); eliminations `r2c3 ≠ 7` via `FutoRules-V2.1/GENERAL/nrc-output.clp:109`.
- Inequality prints and ordered‑chain helpers for monotonicity constraints:
  - Ascending/descending/hill/valley lines: `asc[k]: r2c1 < r2c2 < ... ==>`, `desc[k]: ... ==>`, `hill[k]: ... > ... ==>`, `valley[k]: ... < ... ==>`. Printers in `FutoRules-V2.1/GENERAL/nrc-output.clp:167`–`:231`.
- Chains and subset prints otherwise follow generic notation.
Examples
- Ordered chain: `asc[3]: r2c1 < r2c2 < r2c3 < r2c4 ==> r2c4 ≠ 5`
  Plain English: The row must strictly increase; having 5 at r2c4 blocks a valid ordering, so 5 is removed.
- Chain: `whip[3]: r1c1 {2 3} - r1n2 {c1 c4} - c4n2 {r1 .} ==> r1c1 ≠ 2`
  Plain English: Assuming r1c1 = 2 forces an impossible placement further along the chain, so r1c1 ≠ 2.

Kakuro
- Sector/run messages during solve:
  - “horizontal magic sector S‑in‑p, starting in rR cC” and “vertical sector with g‑digs S‑in‑p, starting in rR” while scanning runs and seeding g‑labels/combinations `KakuRules-V2.1/GENERAL/solve.clp:462`, `:1081`–`:1104`.
- g‑labels (combinations/digits) appear via sector templates; chain printing uses generic chain printers (cells shown as `{digit .}` or `{d1 d2}`), while many human‑readable diagnostics reference sums and sector coordinates. See `KakuRules-V2.1/GENERAL/glabels.clp` for g‑comb/g‑dig construction and `.../combinations.clp` for combination lists.
Examples
- Sector message: `horizontal magic sector 16-in-2, starting in r2 c4` (diagnostic while seeding sectors)
  Plain English: A 2-cell run beginning at r2c4 must sum to 16 (digits {7,9}); downstream pruning follows from this.
- Chain: `whip[2]: r3c5 {7 9} - r3n7 {c2 c5} - r3c2 {7 .} ==> r3c5 ≠ 7`
  Plain English: If r3c5 were 7, subsequent forced placements hit a dead end at r3c2; therefore r3c5 ≠ 7.

Hidato / Numbrix
- Values are plain integers laid on an `r/c` grid. Pretty/partial solution prints: `HidatoRules-V2.1/GENERAL/print-partial-sol.clp:34` (grids), and eliminations/assertions use the generic or app output of `rXcY ≠ n` / `rXcY = n`.
- Additional notation is conceptual (distance/topology), not heavily encoded in chain lines; adjacency/distance is enforced in rules rather than shown as special glyphs.
Examples
- Chain: `whip[2]: r2c3 {7 9} - r2n7 {c1 c3} - r2c1 {7 .} ==> r2c3 ≠ 7`
  Plain English: Assuming r2c3 = 7 forces an impossible placement at r2c1; reject 7 from r2c3.
- Elimination: `r8c5 ≠ 14`
  Plain English: 14 at r8c5 violates adjacency/distance rules given current values, so it’s removed.

Slitherlink
- Label format and value symbols include an edge type prefix: `H`, `V`, `I`, `P`, `B` followed by `r/c` coordinates; value names depend on type (e.g., degree/line presence). `print-label` `SlitherRules-V2.1/GENERAL/nrc-output.clp:16`.
- Bivalue/final cells: `<Type rRcC> {v1 v2}` and `<Type rRcC> {v .}` `SlitherRules-V2.1/GENERAL/nrc-output.clp:28`, `:50`.
- Eliminations: `<Type rRcC> ≠ <value>` `SlitherRules-V2.1/GENERAL/nrc-output.clp:80`.
- Some family‑specific print lines (e.g., degree rules) include explicit type/value names in the message text under `GENERAL/S.clp`.
Examples
- Chain: `biv-chain[2]: V r2c3 {0 1} - V r5c3 {0 1} ==> V r2c3 ≠ 1`
  Plain English: Turning on the vertical edge at (r2,c3) would force an illegal loop/degree configuration; it must be off.
- Elimination: `H r3c5 ≠ 1`
  Plain English: The horizontal edge at (r3,c5) can’t be on without breaking Slitherlink constraints, so set it to 0.

Map Colouring
- Label format: `<Country><Colour>` (e.g., `C12 B`) with names resolved by `country-name`/`colour-name`: `MapRules-V2.1/GENERAL/output.clp:16`.
- Bivalue/final cells: `Country {Colour1 Colour2}` and `Country {Colour .}` `MapRules-V2.1/GENERAL/output.clp:28`, `:40`.
- Eliminations: `Country ≠ Colour` `MapRules-V2.1/GENERAL/output.clp:55`.
- Neighbourhood constraints are implicit; if printed, they are described as “neighbour” in init‑links context (but not embedded in the label text).
Examples
- Chain: `biv-chain[3]: France {Blue Red} - Germany {Red Green} - Czechia {Green .} ==> France ≠ Red`
  Plain English: If France were Red, neighbours would be forced into a conflict by the end of the chain; so France ≠ Red.
- Elimination: `Brazil ≠ Yellow`
  Plain English: Yellow for Brazil clashes with neighbour colours; it’s eliminated.

General Reminders
- Chain headers report the family and length: `whip[n]`, `biv-chain[n]`, `g-whip[n]`, `braid[n]`, typed variants as `biv-chain-<type>[n]`.
- Steps are rendered as `{left right}` pairs in the CSP variable context (rc/rn/cn/bn or app type); links are ` - `, and the final implication is ` ==> not <label>` (or `==> assert <label>` for forcing variants). ORk variants may print `||` side branches but produce a single conclusion per pattern.

<a id="excerpts"></a>
**End‑to‑End Excerpts (from Examples)**

Sudoku — Metcalf‑B7B (CSP-Rules-Examples/Sudoku-a/Metcalf-B7B.clp)
- `whip[2]: b3n1{r1c7 r3c9} - b3n2{r3c9 .} ==> r1c7 ≠ 4, r1c7 ≠ 6`
  Plain English: If block 3’s 1 were at r1c7, block 3’s 2 would be forced impossibly via r3c9; so r1c7 cannot be 4 or 6.
- `hidden-single-in-a-block ==> r8c9 = 3`
  Plain English: Only r8c9 can take 3 in its block; assert it.
- `whip[5]: r6n3{c1 c5} - r4n3{c5 c8} - c8n5{r4 r3} - r2n5{c7 c6} - r2n3{c6 .} ==> r5c1 ≠ 3`
  Plain English: Assuming r5c1 is 3 forces a contradiction down the chain; remove 3 from r5c1.
- `naked-single ==> r9c3 = 2`
  Plain English: r9c3 has one candidate left; set it to 2.
- `braid[7]: r7n2{c6 c5} - r7n1{c5 c9} - r7n6{c9 c8} - r3c8{n6 n5} - r3c9{n1 n2} - r3c6{n9 n6} - r1n6{c5 .} ==> r7c6 ≠ 9`
  Plain English: A long braid shows 9 at r7c6 leads to inconsistency; eliminate it.

Futoshiki — 9x9‑Extreme‑SHT (CSP-Rules-Examples/Futoshiki/Tatham/9x9-Extreme-SHT.clp)
- `asc[3]: r2c9<r1c9<r1c8<r2c8 ==> r2c9 ≠ 9, r2c9 ≠ 8, r2c9 ≠ 7, ...`
  Plain English: A strictly increasing chain across these cells rules out many values to preserve ordering.
- `asc[2]: r7c6<r7c7<r6c7 ==> r7c7 ≠ 1, r7c6 ≠ 8, r6c7 ≠ 2, ...`
  Plain English: Another `<` segment prunes candidates to keep the increase feasible.
- `hidden-single-in-a-row ==> r4c2 = 9`
  Plain English: In row 4, only c2 can take 9; set it.
- `naked-single ==> r9c5 = 4`
  Plain English: Cell r9c5 is forced to 4 by previous eliminations.

<a id="see"></a>
**See Also**
- Trigger: taxonomy of patterns that generate these lines — [Trigger](Trigger.md)
- Model: how labels/cells are formed — [Model](Model.md)
- Overview: where print helpers live — [Overview](Overview.md)
- State: counters and density printed around traces — [State](State.md)
- Beyond: how typed variables affect printed cells — [Beyond](Beyond.md)
- T&E: how hypotheses appear in logs — [T&E](T&E.md)
