## Contents
- [Plain Mental Model](#plain)
- [Technical Model](#technical)
- [Node Types](#nodes)
- [Edge Types](#edges)
- [How Graphs Are Built](#build)
- [How Graphs Evolve (Pruning)](#pruning)
- [Typed, Grouped, and Derived Variants](#variants)
- [Acyclic? Weighted? Connected?](#properties)
- [What To Ask For Your App](#questions)
- [Pointers](#pointers)

<a id="plain"></a>
**Plain Mental Model**
- A candidate is a possible value for a variable. Think “tiny fact card” that says “row 4, col 7 could be digit 2” in Sudoku.
- The solver draws an undirected graph where each node is one such candidate. An edge means those two can’t both be true at the same time (they “see” each other by a constraint), so the graph is a conflict graph.
- Chains/whips/braids are walks in this graph with extra structure. They alternate through linked candidates and the variable each belongs to, and if a walk proves the target would force a contradiction, the target is eliminated.
- Some families also add a grouped layer of nodes (g‑candidates) that represent a structured group (e.g., a row‑segment in Sudoku, or a Kakuro sector combination). Edges between candidates and these g‑nodes are “glinks”.
- Edges are computed once when serious solving starts; candidates get removed as deductions happen. The graph itself is not physically rebuilt on every elimination; rules pattern‑match only active candidates.

<a id="technical"></a>
**Technical Model**
- Nodes:
  - `candidate` facts with `status` `cand` or `c-value` and a unique integer `label`. Generic template: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:79`.
  - Optional `g-candidate` facts for grouped reasoning with their own `label` and `type`. Generic template: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:101`.
- Edges (facts and helpers):
  - Candidate–candidate links: `exists-link ?cont ?lab1 ?lab2` and `csp-linked ?cont ?lab1 ?lab2 ?csp`. Built in `init-links`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:98` and `:118`.
  - Candidate–g‑candidate links: `exists-glink ?cont ?lab ?glabel` and `csp-glinked ?cont ?lab ?glabel ?csp`. Built in app `init-glinks` (e.g., Sudoku): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp:128` and `:140`.
  - App‑level adjacency predicate `labels-linked` defines when two labels conflict (edges): Sudoku example: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:430`.
  - Generic helpers/cache for edges: `linked`, `add-link`, `glinked`, `add-glink` using global caches `?*links*`/`?*glinks*`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:216`, `:230`, `:255`, `:273`.

<a id="nodes"></a>
**Node Types**
- `candidate` (generic): has `context`, `status` (`cand` or `c-value`), `label`, `flag`. Template: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:79`.
- App‑extended `candidate` (e.g., Sudoku): adds structural slots like `number`, `row`, `column`, `block`, `square`: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp:34`.
- `g-candidate` (generic): `context`, `label`, `type`, optional `csp-var`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:101`. Apps extend with typed fields; Sudoku adds `number`, `row/column`, `block`: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp:54`.
- Structural variables tie candidates to CSP variables: `is-csp-variable-for-label` and typed variants, used both for csp‑linked edges and typed chains: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:112` and `:124`.

What a node “holds” (practically)
- A unique `label` identifying a modeled position/value.
- App structure for fast matching (e.g., Sudoku’s row/col/block) enabling direct adjacency tests.
- `context` for T&E/DFS branching; facts are duplicated per context.
- Status: `cand` participates in graph reasoning; `c-value` is a decided value.

<a id="edges"></a>
**Edge Types**
- Ordinary link (candidate–candidate): `exists-link`. Asserted bidirectionally once per pair; counted for density. Built by `init-effective-non-csp-links`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:118` and asserted at `:129` and `:130`.
- CSP link (same CSP variable): `csp-linked ?cont ?lab1 ?lab2 ?csp`. From `is-csp-variable-for-label`; built in `init-effective-csp-links`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:98` and asserted at `:111`.
- Glink (candidate–g‑candidate): `exists-glink` and `csp-glinked` in app init; Sudoku `init-glinks` rules: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp:128` and `:140`.
- App adjacency predicate: `labels-linked(?lab1,?lab2)` encodes constraints that imply mutual exclusion; Sudoku’s definition combines rc/rn/cn/bn conflicts: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:430`.
- Derived adjacencies used by families:
  - `biwhip-linked`/`bibraid-linked` augment `linked` with known contradictory pairs kept in globals like `?*all-biwhip-contrads*`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:326` and `:352`.
  - `g2-linked` supports g2‑whips (a candidate linked to a pair of rlc’s): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:290`.

Edge semantics per app
- Apps override `labels-linked` to define when two labels are adjacent. Example (Sudoku): two candidates are linked if they are in the same cell with different digits, or same row/column/block with same digit: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:441`–`:446`.
- Apps may redefine `linked` to directly call `labels-linked` for speed instead of using the global link cache; Sudoku does: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:487`.

<a id="build"></a>
**How Graphs Are Built**
- Timing: Links are activated after Singles/ECP (BRT) and before `play` begins. Rule `activate-init-links` asserts `(init-links ?cont)` once BRT has run: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:49`.
- Ordinary links:
  - `init-effective-csp-links` scans pairs of candidates sharing a CSP variable to assert `csp-linked`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:98`.
  - `init-effective-non-csp-links` scans candidate pairs where `labels-linked` holds to assert `exists-link` (bidirectional) and update the link cache via `add-link` in context 0: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:118` and `:131`; cache updater `add-link`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:230`.
- Grouped layer (when enabled):
  - App rules assert `g-candidate` facts when a structural group is populated (e.g., Sudoku’s 2D segments): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp:104`.
  - App rules assert `csp-glinked` and `exists-glink` between candidates and g‑candidates (and update the glink cache via `add-glink` in context 0): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp:137` and `:140`; cache updater `add-glink`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:273`.
- Density: when `play` is asserted, density is computed with the standard undirected‑graph formula on active candidate/edge counts: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/play.clp:42`, `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/utils.clp:202`.

<a id="pruning"></a>
**How Graphs Evolve (Pruning)**
- Edges are not retracted when candidates are eliminated; chain/subset/uniqueness rules match only on active candidates. Generic note: link helpers “don’t have to be updated when a candidate is deleted”: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:212`.
- Eliminations physically retract `candidate` facts; e.g., `whip[6]` finds a chain using `exists-link`/`csp-linked` and then retracts the target candidate `(status cand)`: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-SPEED/WHIPS/Whips[6].clp:72` and `:83`.
- During T&E/DFS, link facts are recomputed inside each child context by re‑running the same init‑links phase; globals like the link cache are context‑0 only. See T&E doc (links recomputed per child): `Docs/T&E.md:40`.

Practical upshot
- Think “persistent edges over the current set of active nodes.” As solving proceeds, the node set shrinks and chain walks naturally avoid removed nodes because rules require `(candidate ... (status cand))` for targets and anchors.

<a id="variants"></a>
**Typed, Grouped, and Derived Variants**
- Typed chains restrict steps to specific `csp-var-type` classes using `is-typed-csp-variable-for-label`/`glabel` and `csp-linked`: templates at `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:124` and app mappings, e.g., Sudoku’s `csp-var-type`: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:252`.
- Grouped (g‑) variants add g‑candidates and candidate↔glabel glinks to the traversal space; initialization shown above in Sudoku `init-glinks`.
- Bi‑whips/Bi‑braids augment adjacency with discovered contradictory pairs kept in globals (`?*all-biwhip-contrads*`, `?*all-bibraid-contrads*`): implementations at `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:326` and `:352`.
- g2‑whips use `g2-linked` to treat a candidate linked to two rlc’s as a single logical step: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:290`.
- ORk/forcing families thread specific OR relations into the same graph walks; these are additional structured constraints layered on top of the link/glink graph (see families under `CSP-Rules-Generic/CHAIN-RULES-EXOTIC/*`).

<a id="gchains"></a>
**Deeper: g‑Chains**
- What they traverse: the ordinary candidate link graph plus bipartite glinks to g‑candidates. Rules check `exists-link` and `exists-glink` when expanding a chain.
- Construction recap (Sudoku):
  - Build 2D segments and g‑candidates: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp:38` (init), `:104` (g-candidate horiz), `:128` (csp-glink rn), `:146` (bn), `:164` (cn), `:182` (bn for verti)
  - Glinks asserted as both `csp-glinked` and `exists-glink`; cache updated in context 0 via `add-glink`: `.../init-glinks.clp:140`, `:158`, `:176`, `:194`
- Using g‑chains: g‑whips/g‑braids live under generic chain rules; they treat a g‑candidate step as an alternate to a label step, preserving the alternating logic of whips/braids. Enable via g‑families in config; apps may restrict availability (e.g., Latin Squares disables g‑chains; see config remark: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1-config.clp:136`).
- Mental model: g‑labels are “group abstraction nodes.” Edges to them summarize constraints among many labels (e.g., sector sum feasibility in Kakuro, segment membership in Sudoku). Chains can jump into a group, then back to a label that conflicts with the group’s content.

<a id="typed"></a>
**Deeper: Typed Subgraphs (2D/mono‑typed chains)**
- Typed edges: in addition to `csp-linked`, Sudoku asserts `typed-csp-linked` per CSP type during init‑links, gated by `?*Typed-Chains*` and optional type restrictions:
  - `rc`: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-links.clp:31` (assert `typed-csp-linked ... rc`)
  - `rn`: `.../init-links.clp:55` (assert `... rn`)
  - `cn`: `.../init-links.clp:79` (assert `... cn`)
  - `bn`: `.../init-links.clp:103` (assert `... bn`)
- Chain matching: typed chain rules match a `typed-chain` head that records `csp-type` and constrain expansion with `is-typed-csp-variable-for-label` plus the typed csp adjacency. Example elimination rule:
  - `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-MEMORY/TYPED-Z-CHAINS/Typed-z-chains[12].clp:120` (typed-chain head, `is-typed-csp-variable-for-label`, and use of `typed-csp-linked`)
- Seeding typed facts for labels: during solve/init, apps assert typed mappings per label for all types they want to allow; Sudoku examples:
  - `rc/rn/cn/bn` assertions: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp:127`, `:133`, `:139`, `:146` (and the inverse order later: `:197`, `:203`, `:209`, `:215`)
- Mental model: a typed subgraph is a filtered view of the link graph where only edges and steps consistent with a chosen CSP type are considered. Typed chains walk strictly within that subgraph.

<a id="ork"></a>
**Deeper: ORk Relations and Forcing Chains**
- ORk facts: An OR fragment is materialized as an `ORk-relation` with slots like `OR-name`, `OR-size`, `OR-candidates` and `context`. Tridagon modules are a primary source of such relations in Sudoku (see `SudoRules-V20.1/EXOTIC/Tridagons/*` that assert `ORk-relation`).
- Forcing Whips (ORk): Rules combine the ordinary link graph and an OR fragment. Example, k=2 elimination when a candidate is linked to both branches:
  - `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-EXOTIC/OR2-FORCING-WHIPS/OR2-Forcing-Whips[1].clp:26` (activation) and `:33` (matching `(ORk-relation ... (OR-size 2) (OR-candidates ?zzz1 ?zzz2))` and `exists-link` to both) leading to a retraction of the target candidate.
- Variants and ordering: Generic loader includes OR2..OR8 Forcing Whips and their g‑variants; salience allows choosing whether forcing runs before plain ORk whips (see `CSP-Rules-Generic/GENERAL/saliences.clp` entries conditioned by `?*ORk-Forcing-Whips-before-ORk-Whips*`).
- Mental model: an ORk fragment is a mini‑substructure saying “at least one of these candidates is true.” ORk chains thread normal chain segments before/after this fragment; forcing versions eliminate any candidate attacked by all branches of the OR (or assert a candidate supported by all branches).

<a id="properties"></a>
**Acyclic? Weighted? Connected?**
- Underlying link/glink graphs are undirected and may contain cycles; they are not DAGs. Edges are asserted symmetrically for each pair: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:129` and `:130`.
- Graphs are unweighted. If multiple constraints link a pair, `exists-link` is asserted once (duplicates avoided) and `labels-linked-by` can tell which constraint(s) apply: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:125` and `SudoRules background.clp:407`.
- Connectivity varies with the puzzle state. Density is reported at the start of `play` as a global summary: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/play.clp:42`.

<a id="questions"></a>
**What To Ask For Your App**
- What exactly counts as a link? Implement/inspect `labels-linked` and, if needed, `labels-linked-by` in your app’s `GENERAL/background.clp`.
- Do you need typed chains? Define `csp-var-type` and assert `is-typed-csp-variable-for-label` facts during solve/load.
- Do grouped patterns help? Define `g-candidate`s, `is-csp-variable-for-glabel`, and app `init-glinks` rules; decide what glinks (`physical-glink`/`physical-csp-glink`) should exist.
- Should `linked` use the global cache or be a direct `labels-linked` call? Sudoku redefines it for speed.
- How will contradictions be fed back into adjacency? Consider bi‑whip/bi‑braid contradiction sets if those families are enabled.
- How will T&E/DFS contexts be handled? Ensure child contexts recompute link facts; don’t rely on global caches inside children.

<a id="pointers"></a>
**Pointers**
- Generic templates and link/glink helpers: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`, `.../generic-background.clp`.
- Link activation and density: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`, `.../play.clp`, `.../utils.clp` (`density`).
- Sudoku adjacency (edge semantics): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`.
- Sudoku grouped layer: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp`.
- Chain rules that consume edges: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-*/*` (see `Whips[6]` for a complete example at `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-SPEED/WHIPS/Whips[6].clp:72`).

<a id="apps"></a>
**Per‑App Graph Models**

Kakuro (KakuRules)
- Edges (labels-linked): two candidate labels are linked if they share the same row within the same horizontal sector or the same column within the same vertical sector (and are different rc‑cells). See `nrchv-linked` and `labels-linked`:
  - `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/background.clp:145` (rc-linked semantics helper)
  - `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/background.clp:187` (nrchv-linked)
  - `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/background.clp:224` (labels-linked)
- Grouped layer (sectors, combinations and digits):
  - g‑combinations per sector, creating g‑labels at the controller and glinks to excluded digits in sector cells:
    - Horizontal sectors: `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/glabels.clp:32` (init), `:49` (deal-with-horizontal-gcombs)
    - Vertical sectors: `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/glabels.clp:86` (deal-with-vertical-gcombs)
  - g‑digits per sector cell, creating g‑labels for “one cell’s digit set” and glinks to controller combinations they exclude:
    - Horizontal gdigs: `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/glabels.clp:131` (deal-with-horizontal-gdigs)
    - Vertical gdigs: `CSP-Rules/CSP-Rules-V2.1/KakuRules-V2.1/GENERAL/glabels.clp:264` (deal-with-vertical-gdigs)
- Mental model: candidate nodes are cell–digit labels; edges enforce “same sector run all‑different” and sector structure. g‑labels encode feasible combination sets per sector and feasible digit sets per cell; glinks connect candidates to these sets to propagate sums/compatibility.

Futoshiki (FutoRules)
- Edges (labels-linked): combination of row/column (nrc) conflicts and explicit inequality arcs between adjacent cells:
  - NRC adjacency: `labels-nrc-linked` via same cell with different digits or same row/column same digit: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/background.clp:312`
  - Inequality adjacency: `labels-ineq-linked` consults `?*labels-ineq-links*` populated by `add-labels-ineq-link`: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/background.clp:332` and `:356`
  - Combined: `labels-linked` returns NRC or inequality links: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/background.clp:339`
- Grouped layer (number segments for inequalities):
  - G‑labels represent number segments per cell: “k+” (>= k) and “k‑” (<= k) with helpers `upper-segment-of`/`lower-segment-of`; glinks are created between label and segment if consistent. See segment and glink predicates:
    - Segment encoding and membership: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/glabels.clp:28`, `:44`, `:57`
    - Label∈glabel tests and glinked/glinked-or overrides: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/glabels.clp:214`, `:281`, `:292`
  - Generic glink initialisation is hooked via app `init-glinks` when g‑chains are enabled: `CSP-Rules/CSP-Rules-V2.1/FutoRules-V2.1/GENERAL/init-glinks.clp:50`, `:65`, `:83`
- Mental model: candidate nodes are cell–digit labels; edges capture both AllDifferent on rows/columns and the directed “<”/“>” order constraints turned into conflicts. g‑segments give coarse ranges per cell and connect to labels to accelerate inequality propagation in g‑chains.

Slitherlink (SlitherRules)
- Edges: Slitherlink defines “physical” adjacency offline and imports it, then `init-links` asserts `csp-linked`/`exists-link` from those physical facts:
  - Build from `physical-csp-link`/`physical-link` in precomputed backgrounds: `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/PRECOMPUTED-BACKGROUNDS/background-7x7.clp:2183` (example) and many entries; larger sizes under the same folder.
  - Assertion into working graph: `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/GENERAL/init-links.clp:57` (csp-linked) and `:84` (exists-link)
- Node typing: typed CSP variables encode grid edges/vertices/count/loop parity etc. Single rules operate by type (H, V, N, I, P, B):
  - Typed assertions during solve/load: `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/GENERAL/solve.clp:287` (H), `:306` (V), `:267` (N), `:430` (B) and subsequent lines for related types
  - Specialized Singles per type (degree/loop): `CSP-Rules/CSP-Rules-V2.1/SlitherRules-V2.1/GENERAL/S.clp:41`, `:61`, `:83`, `:105`, `:136`, `:158`, `:180`
- Mental model: candidate nodes represent edge on/off and related typed variables; edges in the graph reflect degree and loop consistency constraints encoded as physical links. The solver consumes these edges via chain/subset families plus type‑specific Singles to enforce the Slitherlink loop rules.

Latin Squares (LatinRules)
- Edges: same rc/rn/cn adjacency as Sudoku, with optional diagonal constraints for pandiagonal variants:
  - Linked‑by dispatcher: `labels-linked-by` with types `rc/rn/cn/dn/an`: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1/GENERAL/background.clp:404`
  - Combined `labels-linked`: rc/rn/cn always; `dn`/`an` included if `?*Pandiagonal*` is true: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1/GENERAL/background.clp:432`
- Grouped layer: Latin Squares do not use g‑chains (no g‑whips/g‑braids) as noted in config: `CSP-Rules/CSP-Rules-V2.1/LatinRules-V2.1-config.clp:136`.
- Mental model: identical conflict graph to Sudoku’s without blocks; optional addition of two diagonal families yields extra adjacency across the main/anti‑diagonals.

Map Colouring (MapRules)
- Nodes: `country×colour` labels per country and colour choice; `country` itself is the CSP variable.
- Edges:
  - Same‑country csp‑links across different colours are asserted as both `csp-linked` and ordinary `exists-link`: `CSP-Rules/CSP-Rules-V2.1/MapRules-V2.1/GENERAL/init-links.clp:45`–`:52`
  - Neighbour constraints become ordinary links between distinct countries with the same colour: `CSP-Rules/CSP-Rules-V2.1/MapRules-V2.1/GENERAL/init-links.clp:64`–`:72`
  - Physical link seeds are produced at solve time (`physical-link` for country and neighbour pairs): `CSP-Rules/CSP-Rules-V2.1/MapRules-V2.1/GENERAL/solve.clp:85`, `:104`, `:189`
- Grouped/typed: no g‑layer; type distinctions are conveyed by the `?country` csp‑var and neighbor tag.
- Mental model: a textbook “country‑colour conflict graph”: each node is a colour assignment; edges encode (a) same country different colours, and (b) neighbours same colour. Chains walk this graph to eliminate colours that would violate these constraints.
