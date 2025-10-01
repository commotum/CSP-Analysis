## Contents
- [Purpose](#purpose)
- [Big Picture](#big-picture)
- [Core Data Model](#core)
- [Labels, Typed Variables, Links](#labels)
- [Phases and Scheduling](#phases)
- [Rule Families](#families)
- [Abstract Data Structure](#abstract)
- [Public API](#api)
- [Config & Flags](#config)
- [Links Drive Rules](#links)
- [Traditional vs CSP‑Rules](#compare)
- [Cheat‑Sheet](#cheat)
- [File Pointers](#files)
- [See Also](#see)

<a id="purpose"></a>
**Purpose**
- Mental model and API of CSP-Rules as used in this workspace.
- Focus on variable types, how rules operate, how links are generated, and how the engine’s components fit together.
- Contrast with “typical” CSP solvers and clarify what’s different here.

<a id="big-picture"></a>
**Big Picture**
- Forward‑chaining rule system (CLIPS + RETE) over a shared working memory of facts.
- Generic core defines data contracts (templates), globals, link initialisation, salience scheduling, and chain families.
- Each application (e.g., Sudoku) extends templates, defines label/variable typing and link semantics, and adds app‑specific rules.
- Execution runs in phases: initialise instance → BRT (Singles + ECP) → init‑links → play (chains, subsets, uniqueness, exotic) until solved.

<a id="core"></a>
**Core Data Model**
- Generic templates
  - `candidate` — atomic candidates with status and label; holds context and flags. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`
  - `g-candidate` — grouped candidate abstraction for g‑chains. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`
  - `csp-variable` — structural variables (typed later by each app). `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`
- Application templates (Sudoku example)
  - Extends `candidate` with Sudoku slots: `number`, `row`, `column`, `block`, `square`, `band`, `stack`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp`
  - Extends `g-candidate` with typed metadata: `type`, `row/column` segments, `block`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp`
- Facts without explicit templates (ordered facts)
  - `csp-linked`, `exists-link` between labels; `csp-glinked`, `exists-glink` for grouped links. Asserted during init‑links/glinks.
  - `context` and `technique` facts orchestrate phases and track the current rule family.

<a id="labels"></a>
**Labels, Typed Variables, and Links**
- Labels and typed CSP variables (Sudoku)
  - Typed variables derive from structure (rows/columns/blocks):
    - `row-number-to-rn-variable` — `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
    - `column-number-to-cn-variable` — `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
    - `block-number-to-bn-variable` — `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
  - Label encoding for (number,row,column): `nrc-to-label` — `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
- Declarative link predicates (app‑level)
  - `labels-linked-by` (by CSP variable type) — `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
  - `labels-linked` (any applicable constraint) — `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
  - These are used both in ECP and for init‑links to derive effective links.
- Effective links creation (generic)
  - Activates after BRT via salience: asserts logical facts for links and counts density.
  - `init-effective-csp-links` asserts `csp-linked` facts for labels sharing a typed `csp-variable`. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`
  - `init-effective-non-csp-links` asserts `exists-link` for any pair where `labels-linked` holds. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`
  - For g‑links (Sudoku): `csp-glinked` and `exists-glink` from candidates to `g-candidate` labels. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp`
- Link sets as globals (fast lookup)
  - `add-link` and `add-glink` push pairs into `?*links*` / `?*glinks*` and update counts. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp`
  - Density computed from candidate and link counts when `play` begins. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/play.clp`

<a id="phases"></a>
**Phases and Scheduling**
- Solve loop (generic model)
  - `solve` initialises globals and app structures, asserts `(context (name 0))`, runs the engine, and prints times. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp`
  - BRT (Singles + ECP) fires first in context 0; after BRT, `init-links` runs; then `(play)` starts non‑trivial rules per salience. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/play.clp`
- Salience definitions coordinate orderings
  - Generic saliences include pre/post BRT, `init-links`, and subsequent phases. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/saliences.clp`
  - Apps may adjust saliences; Sudoku has its own `GENERAL/saliences.clp`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/saliences.clp`
- Technique tracking
  - Many rules assert `(technique ?cont <name>)` and set `?*technique*` for reporting. Rules also decrement `?*nb-candidates*` when eliminating candidates.

<a id="families"></a>
**Rule Families (Resolution Theory)**
- BRT — Basic Resolution Theory
  - ECP (Elementary Constraint Propagation) uses `labels-linked` to prune. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/ECP.clp`
  - Singles: generic scaffolding and app NS/HS implementations (Sudoku). `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/NS.clp`, `.../HS.clp`
- Chains (generic families)
  - Bivalue, z‑chains, t‑whips, whips, g‑whips, braids; speed/memory variants. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-SPEED`, `.../CHAIN-RULES-MEMORY`
  - Exotic ORk variants (forcing/contrad) extend chain reasoning. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-EXOTIC`
- Application‑specific families
  - Sudoku: `SUBSETS/` (Pairs/Triplets/Fish), `UNIQUENESS/` (URs, BUG, Deadly Patterns), `EXOTIC/`, `GOODIES/`, etc. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/SUBSETS`
- Search augmentations
  - T&E(n) and DFS integrated into the same rule framework. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/T&E+DFS`

<a id="abstract"></a>
**Abstract Data Structure**
- Working Memory (facts)
  - Template facts: `candidate`, `g-candidate`, `csp-variable`, and app‑specific templates.
  - Ordered facts: `csp-linked`, `exists-link`, `csp-glinked`, `exists-glink`, `context`, `technique`, etc.
- Global State (defglobals)
  - Configuration/feature flags (enable families, set max lengths, print options). `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp`
  - Performance counters (`?*nb-candidates*`, `?*links-count*`, `?*glinks-count*`, density). `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp`
  - Link caches: `?*links*`, `?*glinks*` as pair lists for fast membership checks in chain rules. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp`
- RETE Network
  - Rules (defrule) compile into a RETE network; pattern matches over facts drive eliminations/assertions without explicit control flow.

<a id="api"></a>
**Public API Surface**
- Load order (REPL)
  - Load a per‑app config; it defines `?*CSP-Rules*`, computes directories, loads generic/app globals, and batches loaders. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp`
- Generic entry (size + list)
  - `(solve <size> <args...>)` initialises and runs the engine, printing times if enabled. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp`
- Sudoku convenience functions (examples)
  - Strings/grids/lists: `solve`, `solve-sudoku-string`, `solve-sudoku-grid`, `solve-sudoku-list`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp`
  - File/batch: `solve-grid-from-text-file`, `solve-n-th...`, `solve-sdk-grid`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve-files.clp`
- Preferences
  - `(solve-w-preferences ...)` delegates to `solve` but allows pref ordering; used in some examples. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/MODULES/modules.clp`

<a id="config"></a>
**Configuration and Feature Flags**
- Chain implementation mode: `?*chain-rules-optimisation-type*` = `SPEED` or `MEMORY`. Default is SPEED. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp`
- Enable/disable families: `?*Subsets*`, `?*Whips*`, `?*Braids*`, typed/g‑variants, ORk options. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp`
- Max lengths per family: whips/braids/typed/g/forcing/ORk. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/ADVANCED/disable-re-enable-rules.clp`
- App‑specific toggles (Sudoku): `?*FinnedFish*`, `?*Unique-Rectangles*`, `?*BUG*`, `?*Deadly-Patterns*`, Tridagons, etc. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp`
- Print options: `?*print-actions*`, `?*print-levels*`, `?*print-solution*`, summaries after Singles/final RS. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp`

<a id="links"></a>
**How Links Drive Rules**
- Chain rules and many app rules match on `exists-link`/`csp-linked`/`exists-glink` to walk adjacency among candidates and g‑candidates.
- Typed reasoning leverages `csp-linked` with specific typed variables, while general reasoning uses `exists-link`.
- g‑chains rely on `exists-glink` plus app‑specific grouping of labels into `g-candidate`s (e.g., row/column segments in Sudoku). `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp`

<a id="compare"></a>
**How CSPs Normally Work vs CSP‑Rules**
- Typical CSP solvers
  - Variables with finite domains; constraints propagate (AC/GAC) and search/backtracking explores assignments.
  - Data structures: domain lists, constraint graphs; algorithms like AC‑3/4, MAC, CBJ; search with heuristics.
- CSP‑Rules (different approach)
  - Forward‑chaining production system: constraints and techniques encoded as rules triggered by facts, not pull‑style propagators.
  - Pattern‑based resolution: families like whips/braids encode human‑readable elimination patterns; chains traverse `exists-link`/`csp-linked`.
  - Structural modeling: introduces additional typed CSP variables (e.g., `rn/cn/bn` in Sudoku) to uniformise constraints and enable generic techniques.
  - Integrated T&E/DFS: optional, controlled by flags, but implemented within the same rule framework rather than a separate solver.
  - Single stream of reasoning within each pattern (e.g., whips), even when exploring ORk variants.

<a id="cheat"></a>
**Mental Model Cheat‑Sheet**
- State = facts (candidates, csp‑vars, links) + globals (toggles, counters, caches).
- Engine = RETE network over defrules with saliences; rules assert `(technique ...)`, eliminate candidates (set to `c-value` or retract), and add derived links.
- Adjacency = `labels-linked` (app) → `exists-link` (generic) → chains and subsets traverse it.
- Grouping = `g-candidate` + `exists-glink` → g‑variants of chains.
- Phases = BRT → init‑links → play (chains/subsets/uniqueness/exotic) until solved or limits.
- API = load config → generic loader → app loader → call `solve`/app helpers.

<a id="files"></a>
**File Pointers (Quick Index)**
- Generic templates: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`
- Sudoku templates: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp`
- Sudoku background (labels, rn/cn/bn, links): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`
- Generic init‑links: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`
- Sudoku init‑glinks: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp`
- Generic solve (model): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp`
- Sudoku solve helpers: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp` (`solve`, `solve-sudoku-string`, `solve-sudoku-list`, `solve-sudoku-grid`)
- Salience setup: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/saliences.clp`
- Chain families: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-SPEED`, `.../CHAIN-RULES-MEMORY`, `.../CHAIN-RULES-EXOTIC`

<a id="see"></a>
**See Also**
- Overview: architecture, run order — [Overview](Overview.md)
- State: working memory + globals — [State](State.md)
- Pattern taxonomy: how rules read as Scope → Trigger → Action — [Trigger](Trigger.md)
- Output notation: how chains print — [Notation](Notation.md)
- Non‑binary → binary across apps — [Beyond](Beyond.md)
- Trial & Error and DFS — [T&E](T&E.md)
