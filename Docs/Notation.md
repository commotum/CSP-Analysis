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

**Label And Cell Printing (Sudoku example)**
- Label formatting (one candidate):
  - Sudoku overrides `print-label` to print number+row+column short names: `5r2c9` `SudoRules-V20.1/GENERAL/nrc-output.clp:24`.
- Bivalue cell (two candidates in one CSP‑variable):
  - Printed as `Block {val1 val2}` depending on CSP type (rc, rn, cn, bn). The value pair appears inside `{}` separated by a space: `nrc-output.clp:45` and `:69`.
- Final cell in non‑reversible chains (whips/braids):
  - Printed with a dot for the missing last right‑linking candidate: `{val .}` `SudoRules-V20.1/GENERAL/nrc-output.clp:88`.

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

**Other Notable Prints**
- Resolution state snapshots
  - After Singles (optional): banner + “Resolution state after Singles” with counts and density. Triggered in `play.clp` if `?*print-RS-after-Singles*` is TRUE.
  - Pretty grid dumps (Latin Squares): `LatinRules-V2.1/GENERAL/pretty-print.clp:86` via `pretty-print-current-resolution-state`.
  - Sudoku partial grid prints: `SudoRules-V20.1/GENERAL/print-partial-sol.clp:34`.
- Banners and timing
  - Start banner and end banner include app/version, engine, rating‑type, and machine descriptor; times printed as `init-time`, `solve-time`, `total-time` when `?*print-time*`/`?*print-actions*` are enabled. See `CSP-Rules-Generic/GENERAL/solve.clp:20` (`print-start-banner`, `print-banner`).
- Level/phase traces
  - When `?*print-levels*` is TRUE, rules announce entry into higher‑level families (e.g., “Entering_level_tridagon[12]”). These are family‑specific debug prints.

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

