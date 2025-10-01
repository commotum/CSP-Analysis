**Purpose**
- Mental model and API of CSP-Rules as used in this workspace.
- Focus on variable types, how rules operate, how links are generated, and how the engine’s components fit together.
- Contrast with “typical” CSP solvers and clarify what’s different here.

**Big Picture**
- Forward‑chaining rule system (CLIPS + RETE) over a shared working memory of facts.
- Generic core defines data contracts (templates), globals, link initialisation, salience scheduling, and chain families.
- Each application (e.g., Sudoku) extends templates, defines label/variable typing and link semantics, and adds app‑specific rules.
- Execution runs in phases: initialise instance → BRT (Singles + ECP) → init‑links → play (chains, subsets, uniqueness, exotic) until solved.

**Core Data Model**
- Generic templates
  - `candidate` — atomic candidates with status and label; holds context and flags. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:79`
  - `g-candidate` — grouped candidate abstraction for g‑chains. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:101`
  - `csp-variable` — structural variables (typed later by each app). `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:123`
- Application templates (Sudoku example)
  - Extends `candidate` with Sudoku slots: `number`, `row`, `column`, `block`, `square`, `band`, `stack`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp:22`
  - Extends `g-candidate` with typed metadata: `type`, `row/column` segments, `block`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp:41`
- Facts without explicit templates (ordered facts)
  - `csp-linked`, `exists-link` between labels; `csp-glinked`, `exists-glink` for grouped links. Asserted during init‑links/glinks.
  - `context` and `technique` facts orchestrate phases and track the current rule family.

**Labels, Typed Variables, and Links**
- Labels and typed CSP variables (Sudoku)
  - Typed variables derive from structure (rows/columns/blocks):
    - `row-number-to-rn-variable` `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:239`
    - `column-number-to-cn-variable` `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:243`
    - `block-number-to-bn-variable` `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:247`
  - Label encoding for (number,row,column): `nrc-to-label` `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:294`
- Declarative link predicates (app‑level)
  - `labels-linked-by` (by CSP variable type) `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:407`
  - `labels-linked` (any applicable constraint) `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:430`
  - These are used both in ECP and for init‑links to derive effective links.
- Effective links creation (generic)
  - Activates after BRT via salience: asserts logical facts for links and counts density.
  - `init-effective-csp-links` asserts `csp-linked` facts for labels sharing a typed `csp-variable`. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:78`
  - `init-effective-non-csp-links` asserts `exists-link` for any pair where `labels-linked` holds. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:102`
  - For g‑links (Sudoku): `csp-glinked` and `exists-glink` from candidates to `g-candidate` labels. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp:61`
- Link sets as globals (fast lookup)
  - `add-link` and `add-glink` push pairs into `?*links*` / `?*glinks*` and update counts. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:230`, `:273`
  - Density computed from candidate and link counts when `play` begins. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/play.clp:42`

**Phases and Scheduling**
- Solve loop (generic model)
  - `solve` initialises globals and app structures, asserts `(context (name 0))`, runs the engine, and prints times. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp:93`
  - BRT (Singles + ECP) fires first in context 0; after BRT, `init-links` runs; then `(play)` starts non‑trivial rules per salience. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/play.clp:21`
- Salience definitions coordinate orderings
  - Generic saliences include pre/post BRT, `init-links`, and subsequent phases. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/saliences.clp:448`
  - Apps may adjust saliences; Sudoku has its own `GENERAL/saliences.clp`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/saliences.clp:1`
- Technique tracking
  - Many rules assert `(technique ?cont <name>)` and set `?*technique*` for reporting. Rules also decrement `?*nb-candidates*` when eliminating candidates.

**Rule Families (Resolution Theory)**
- BRT — Basic Resolution Theory
  - ECP (Elementary Constraint Propagation) uses `labels-linked` to prune. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/ECP.clp:1`
  - Singles: generic scaffolding and app NS/HS implementations (Sudoku). `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/NS.clp:1`, `.../HS.clp:1`
- Chains (generic families)
  - Bivalue, z‑chains, t‑whips, whips, g‑whips, braids; speed/memory variants. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-SPEED:1`, `.../CHAIN-RULES-MEMORY:1`
  - Exotic ORk variants (forcing/contrad) extend chain reasoning. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-EXOTIC:1`
- Application‑specific families
  - Sudoku: `SUBSETS/` (Pairs/Triplets/Fish), `UNIQUENESS/` (URs, BUG, Deadly Patterns), `EXOTIC/`, `GOODIES/`, etc. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/SUBSETS:1`
- Search augmentations
  - T&E(n) and DFS integrated into the same rule framework. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/T&E+DFS:1`

**Abstract Data Structure**
- Working Memory (facts)
  - Template facts: `candidate`, `g-candidate`, `csp-variable`, and app‑specific templates.
  - Ordered facts: `csp-linked`, `exists-link`, `csp-glinked`, `exists-glink`, `context`, `technique`, etc.
- Global State (defglobals)
  - Configuration/feature flags (enable families, set max lengths, print options). `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp:448`
  - Performance counters (`?*nb-candidates*`, `?*links-count*`, `?*glinks-count*`, density). `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp:98`
  - Link caches: `?*links*`, `?*glinks*` as pair lists for fast membership checks in chain rules. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/generic-background.clp:210`
- RETE Network
  - Rules (defrule) compile into a RETE network; pattern matches over facts drive eliminations/assertions without explicit control flow.

**Public API Surface**
- Load order (REPL)
  - Load a per‑app config; it defines `?*CSP-Rules*`, computes directories, loads generic/app globals, and batches loaders. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp:49`, `:87`, `:90`, `:890`
- Generic entry (size + list)
  - `(solve <size> <args...>)` initialises and runs the engine, printing times if enabled. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp:125`
- Sudoku convenience functions (examples)
  - Strings/grids/lists: `solve`, `solve-sudoku-string`, `solve-sudoku-grid`, `solve-sudoku-list`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp:500`, `:469`, `:1184`, `:861`
  - File/batch: `solve-grid-from-text-file`, `solve-n-th...`, `solve-sdk-grid`. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve-files.clp:63`, `:99`, `:430`
- Preferences
  - `(solve-w-preferences ...)` delegates to `solve` but allows pref ordering; used in some examples. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/MODULES/modules.clp:98`

**Configuration and Feature Flags**
- Chain implementation mode: `?*chain-rules-optimisation-type*` = `SPEED` or `MEMORY`. Default is SPEED. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp:360`
- Enable/disable families: `?*Subsets*`, `?*Whips*`, `?*Braids*`, typed/g‑variants, ORk options. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp:399`, `:448`
- Max lengths per family: whips/braids/typed/g/forcing/ORk. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/ADVANCED/disable-re-enable-rules.clp:344`
- App‑specific toggles (Sudoku): `?*FinnedFish*`, `?*Unique-Rectangles*`, `?*BUG*`, `?*Deadly-Patterns*`, Tridagons, etc. `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp:113`, `:119`, `:156`
- Print options: `?*print-actions*`, `?*print-levels*`, `?*print-solution*`, summaries after Singles/final RS. `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp:1801`

**How Links Drive Rules**
- Chain rules and many app rules match on `exists-link`/`csp-linked`/`exists-glink` to walk adjacency among candidates and g‑candidates.
- Typed reasoning leverages `csp-linked` with specific typed variables, while general reasoning uses `exists-link`.
- g‑chains rely on `exists-glink` plus app‑specific grouping of labels into `g-candidate`s (e.g., row/column segments in Sudoku). `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp:22`

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

**Mental Model Cheat‑Sheet**
- State = facts (candidates, csp‑vars, links) + globals (toggles, counters, caches).
- Engine = RETE network over defrules with saliences; rules assert `(technique ...)`, eliminate candidates (set to `c-value` or retract), and add derived links.
- Adjacency = `labels-linked` (app) → `exists-link` (generic) → chains and subsets traverse it.
- Grouping = `g-candidate` + `exists-glink` → g‑variants of chains.
- Phases = BRT → init‑links → play (chains/subsets/uniqueness/exotic) until solved or limits.
- API = load config → generic loader → app loader → call `solve`/app helpers.

**File Pointers (Quick Index)**
- Generic templates: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp:79`
- Sudoku templates: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp:22`
- Sudoku background (labels, rn/cn/bn, links): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:239`, `:294`, `:407`, `:430`
- Generic init‑links: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:49`
- Sudoku init‑glinks: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/init-glinks.clp:22`
- Generic solve (model): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp:125`
- Sudoku solve helpers: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp:469`, `:500`, `:861`, `:1184`
- Salience setup: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/saliences.clp:448`
- Chain families: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-SPEED:1`, `.../MEMORY:1`, `.../EXOTIC:1`

**See Also**
- Overview: architecture, run order — [Overview](Overview.md)
- State: working memory + globals — [State](State.md)
- Pattern taxonomy: how rules read as Scope → Trigger → Action — [Trigger](Trigger.md)
- Output notation: how chains print — [Notation](Notation.md)
- Non‑binary → binary across apps — [Beyond](Beyond.md)
- Trial & Error and DFS — [T&E](T&E.md)
