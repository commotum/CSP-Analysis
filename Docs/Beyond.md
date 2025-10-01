## Contents
- [Idea At A Glance](#idea)
- [Instances Across Applications](#instances)
- [Similarities vs Differences](#compare)
- [API Surface](#api)
- [Why This Works](#why)
- [Design Checklist](#design)
- [Quick File Index](#files)
- [See Also](#see)

<a id="idea"></a>
**Beyond Binary: How CSP‑Rules Encodes Non‑Binary Constraints**

**Idea At A Glance**
- Reference note: Pointers use file paths and named symbols, not line numbers, to avoid drift across versions.
- Model every global/non‑binary constraint by introducing extra, typed CSP variables whose domains enumerate the local choices that must be mutually exclusive.
- Connect those typed variables to the concrete candidate labels via `is-csp-variable-for-label` facts; then assert pairwise contradiction links (`csp-linked`) between labels that share the same typed variable.
- Optionally add non‑csp links (`exists-link`) for relational constraints that aren’t “exactly‑one” (e.g., inequalities, adjacency, distance).
- Chain rules and Singles then operate over a pure graph of binary links and glinks.

Core machinery (generic)
- `csp-variable`, `candidate`, `g-candidate` templates: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`
- Mapping facts: `is-csp-variable-for-label (csp-var …) (label …)` produced per app and used by init‑links to assert `csp-linked` pairs: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`
- Application “link semantics”: `labels-linked-by`, `labels-linked` determine which label pairs are related by a typed variable vs any constraint: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp`
- Effective links creation: `init-links` asserts binary links and counts density: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`

What “extra variables” buy you
- Non‑binary constraints (AllDifferent, sum‑in‑run, adjacency, etc.) are “binaryized” as either:
  - csp‑links: pairs of labels that share a typed CSP variable; or
  - exists‑links: pairs related by non‑csp relations (inequality, neighbour, distance).
- This uniform link graph powers all chain families (whips, braids, typed/g‑variants) and Singles.

<a id="instances"></a>
**Instances Across Applications**

Sudoku (SudoRules)
- Extra variables: `rc`, `rn`, `cn`, `bn` map (row,col), (row,number), (col,number), (block,number).
  - Constructors: `row-column-to-rc-variable`, `row-number-to-rn-variable`, `column-number-to-cn-variable`, `block-number-to-bn-variable`: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
  - Type introspection: `csp-var-type` returns one of `rc/rn/cn/bn`: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
- Binaryization
  - `labels-linked-by` dispatches to `rc/rn/cn/bn` link tests; `labels-linked` ORs them: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
  - Generic `init-links` asserts `csp-linked` between labels sharing a typed variable; `exists-link` for application links: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`
- Effect: AllDifferent over row/col/block and per‑cell exclusivity become pairwise `csp-linked` constraints; chains walk this graph.

Latin Squares (+ Pandiagonals)
- Same base variables: `rc/rn/cn` and optional diagonal variables `dn` (main) and `an` (anti). Constructors + type mapping: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1/GENERAL/background.clp`
- App overrides generic init‑links to assert both `csp-linked` and `exists-link` per typed family (including diagonals): `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1/GENERAL/init-links.clp`
- `labels-linked` includes diagonal constraints when `?*Pandiagonal*` is enabled: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1/GENERAL/background.clp`
- Effect: AllDifferent over rows, cols, and optional diagonals are binaryized via typed csp variables.

Futoshiki
- Base variables: `rc/rn/cn` like Latin; constructors + type mapping: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/background.clp`
- Non‑csp inequalities: inequality arcs are added as `labels-ineq-links` and folded into `labels-linked`: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/background.clp`
- Effect: AllDifferent constraints are binaryized via `csp-linked`; the “<” constraints become direct `exists-link` relations, coexisting in the same link graph.

Kakuro
- Mixed modeling: white cell digits plus “sector” (run) constraints for sums.
- Natural csp variables: per cell (`rc`) and per run/number (`rn`/`cn`) are created on the fly while scanning sectors; physical csp‑links asserted for both RC and RN constraints: `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/solve.clp`
- Group modeling: sectors with admissible combinations/digits are represented via g‑labels; facts `is-csp-variable-for-glabel` and `physical-csp-glink` connect digits to sector constraints: `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/glabels.clp`
- Binaryization
  - For each run, labels sharing the same run‑typed CSP variable get `csp-linked` (mutual exclusion).
  - Sum feasibility is propagated via glinks between cell labels and sector g‑labels; chain rules traverse `exists-glink` and `exists-link` edges.
- Effect: Global sum constraints reduce to pairwise links plus membership links to sector g‑labels that encode allowed sets.

Map Colouring
- Extra variable: one CSP variable per country (exactly one colour), constructor is identity: `country-to-csp-variable`: `CSP-Rules/CSP-Rules-V2.1/MapRules-V2.1/GENERAL/background.clp`
- Binaryization
  - For a country, all (country,colour) labels are pairwise `csp-linked`: `CSP-Rules/CSP-Rules-V2.1/MapRules-V2.1/GENERAL/init-links.clp`
  - Neighbour constraints become non‑csp links between same‑colour labels of adjacent countries: `CSP-Rules/CSP-Rules-V2.1/MapRules-V2.1/GENERAL/init-links.clp`
- Effect: “one colour per country” is a CSP variable; “neighbours differ” is pairwise excluded by `exists-link`.

Hidato / Numbrix
- Extra variables: `Xrc` (per cell) and `Xn` (per number); type test distinguishes them: `CSP-Rules/CSP-Rules-V2.1/HidatoRules-V2.1/GENERAL/background.clp`
- Binaryization
  - `nrc-linked` implements per‑cell exclusivity and per‑number uniqueness; adjacency/distance feasibility is added as non‑csp links (`distant-linked`): `CSP-Rules/CSP-Rules-V2.1/HidatoRules-V2.1/GENERAL/background.clp`
- Effect: “each number in one cell” and “each cell has one number” are CSP variables; path adjacency is pairwise constraints combining topology or geometry.

Slitherlink
- Typed CSP variables for several structural families (edges, numbers, intersections…): stored by attaching a `csp-var-type` to `is-csp-variable-for-label`: `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/GENERAL/templates.clp`, `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/GENERAL/S.clp`
- Binaryization uses precomputed “physical” links
  - `physical-csp-link` and `physical-link` are materialised first; `init-links` turns them into `csp-linked` and `exists-link`: `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/GENERAL/init-links.clp`
- Effect: local loop/degree constraints are encoded via typed CSP variables; the loop structure is enforced by pairwise links among typed candidates.

Similarities vs Differences
- Similar
  - All apps standardise on: typed CSP variables + `is-csp-variable-for-label` + `init-links` → `csp-linked`/`exists-link`.
  - `labels-linked-by`/`labels-linked` expose a uniform notion of adjacency for chains and Singles.
- Different
  - Which typed variables exist (e.g., Sudoku `bn`, Latin `dn/an`, Hidato `n`, Slither `H/V/N/I/P/B`, Map `country`).
  - Whether non‑csp relations are needed: inequalities (Futoshiki), neighbours (Map), distance (Hidato), sector sums via g‑labels (Kakuro).
  - Some apps precompute “physical” links for performance and clarity (Slitherlink, Kakuro, Map).

**API Surface You’ll See In Code**
- Map labels to typed CSP variables: `is-csp-variable-for-label …` facts (per app); used everywhere to bind labels to variables.
- Typed tests: `csp-var-type` returns symbols used in `labels-linked-by` dispatch.
- Init phase (generic or app‑specific) creates links: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`; app overrides when more efficient or when non‑csp relations are needed (e.g., Latin’s diagonals, Map’s neighbours).
- Optional grouped layer: g‑labels (`g-candidate`), `is-csp-variable-for-glabel`, and glinks connect labels to grouped constraints (Kakuro sectors; Sudoku segments for g‑chains).

**Why This Works (Conceptually)**
- Any n‑ary constraint can be represented by introducing an intermediate variable whose domain is the set of local choices governed by the constraint, and linking all labels that “compete” for that variable.
- Post‑processing adds pairwise relational links for constraints that are not “exactly‑one” (inequality, mutual exclusion across entities, adjacency).
- Once reduced to a labelled graph with binary edges, the same chain/whip/braid machinery applies across domains.

**See Also**
- Overview: high‑level architecture — [Overview](Overview.md)
- Model: labels, typed variables, links — [Model](Model.md)
- Pattern taxonomy: where these links are used — [Trigger](Trigger.md)
- Output notation: how typed cells look — [Notation](Notation.md)
- State: facts and link caches — [State](State.md)
- T&E / DFS: contradictions over csp‑variables — [T&E](T&E.md)

**Design Checklist For New Constraints**
- Identify the global constraint families and choose typed variables (names and encoding) to decompose them.
- Define constructors and `csp-var-type` so `labels-linked-by` can dispatch.
- Assert `is-csp-variable-for-label` for every label/typed variable relation; then let `init-links` emit `csp-linked`.
- Add non‑csp `exists-link` relations for pure relations (inequality, neighbour, distance) that aren’t captured by a typed variable.
- Optionally introduce g‑labels if “group membership” helps (sectors/combinations) and assert `is-csp-variable-for-glabel` and glinks.

**Quick File Index**
- Generic init‑links and background helpers: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`, `.../generic-background.clp`
- Sudoku typed vars + links: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
- Latin typed vars (+ diagonals) + init‑links: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1/GENERAL/background.clp`, `.../init-links.clp`
- Futoshiki inequalities in links: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/background.clp`
- Kakuro sectors + physical links: `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/solve.clp`, `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/glabels.clp`
- Map neighbours + country variables: `CSP-Rules/CSP-Rules-V2.1/MapRules-V2.1/GENERAL/init-links.clp`
- Hidato/Numbrix distance model: `CSP-Rules/CSP-Rules-V2.1/HidatoRules-V2.1/GENERAL/background.clp`
- Slither physical links + typed mapping: `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/GENERAL/init-links.clp`, `.../GENERAL/S.clp`
