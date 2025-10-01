## Contents
- [TL;DR](#tl-dr)
- [Architecture](#architecture)
- [Directory](#directory)
- [Usage Cheat‑Sheet](#usage)
- [What To Read First](#read-first)
- [Data & State Model](#data)
- [Public Surface](#public)
- [Config & Flags](#config)
- [Resolution Theory](#resolution)
- [Naming & Extension](#naming)
- [Comparison](#comparison)
- [Glossary](#glossary)
- [Schema & Flow](#schema)
- [Fast vs Thorough](#fast)

<a id="tl-dr"></a>
**TL;DR**
- This workspace contains a local copy of the CSP-Rules codebase and the companion examples repository: `CSP-Rules/CSP-Rules-V2.1/` and `CSP-Rules-Examples/`.
- CSP-Rules is a pattern-based (rule-based) solver for finite binary CSPs implemented in CLIPS; an application layer (e.g., Sudoku) extends generic templates and rules.
- This document summarizes architecture (generic core → app modules), shows where to look for data/state models and loaders, and captures usage patterns directly from the examples (no execution required).

Reading order (recommended): Overview → Notation → Model → Graphs → State → Trigger → Beyond → T&E. Reason: learn to read traces first, then build mental models, understand the graph (nodes/edges) that rules traverse, and layer in taxonomy and app‑specific details.

Reference note: Pointers in this guide use file paths and named symbols instead of line numbers to avoid drift across versions.

<a id="architecture"></a>
**Architecture At A Glance**
- Generic Core
  - Loader orchestrates generic modules and then batches the active application loader: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CSP-Rules-Generic-Loader.clp`.
  - Templates (data contracts), runtime globals, solve loop and salience scheduling: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`, `.../globals.clp`, `.../solve.clp`, `.../saliences.clp`.
- Link initialization and play loop: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`, `.../play.clp`.
  See also: graph structure and initialization — [Graphs](Graphs.md).
  - Chains and variants (common/exotic; speed/memory): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-*/*`.
  - T&E + DFS augmentations: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/T&E+DFS/`.
- Applications (example: Sudoku)
  - Per‑app loader and modules: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/SudoRules-Loader.clp` (batched by the generic loader), `.../GENERAL/`, `.../SUBSETS/`, `.../UNIQUENESS/`, `.../EXOTIC/`.
  - App templates and background (labels, typed variables, link semantics): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp`, `.../background.clp`.
  - App solve API (selected entries): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp` (`solve`, `solve-sudoku-string`, `solve-sudoku-list`, `solve-sudoku-grid`).
  - Batch/file helpers: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve-files.clp` and friends.
- Configuration
  - Per‑app config defines install path and loaders and can auto-batch the generic loader: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp`.

<a id="directory"></a>
**Directory Organization**
- `CSP-Rules/CSP-Rules-V2.1/`
  - `CSP-Rules-Generic/` — Generic engine (templates, globals, solve, chains, T&E/DFS, utils).
  - `SudoRules-V20.1/` — Sudoku application (GENERAL rules, SUBSETS, UNIQUENESS, EXOTIC, STATS).
  - Other apps — `LatinRules-V2.1/`, `FutoRules-V2.1/`, `KakuRules-V2.1/`, `HidatoRules-V2.1/`, `SlitherRules-V2.1/`, `MapRules-V2.1/`.
  - `CLIPS/` — CLIPS binaries and scripts.
  - `*-config.clp` — Per‑app configuration entrypoints (paths, loaders, toggles).
  - `README.md`, `UPDATES.md`, `Docs/` — High-level docs and manuals.
- `CSP-Rules-Examples/`
  - Example `.clp` files and large‑scale studies for each app (usage patterns, toggles, ratings and timings in comments).
- Root
  - `Docs/Overview.md` — This analysis and guide. `README.md` — Workspace readme.

<a id="usage"></a>
**Examples‑First Usage Cheat‑Sheet (No‑Run)**
- Load Order Convention (from app config and loaders)
  - In CLIPS REPL: load a per‑app config (it sets `?*CSP-Rules*` to your install and defines loader paths), then the config batches the generic loader which in turn batches the app loader.
  - Key config/globals and loader symbols in Sudoku config: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp` (defines `?*CSP-Rules*`, `?*CSP-Rules-Generic-Loader*`, `?*Application-Loader*`, and auto‑batch behaviour).
- Typical Sudoku Invocations (from source)
  - String/grid based: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp` (`(solve "...puzzle...")`, `solve-sudoku-string`, `solve-sudoku-grid`).
  - File/batch helpers: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve-files.clp` (`solve-grid-from-text-file`, `solve-nth-grid-from-text-file`, `solve-n-grids-after-first-p-from-text-file`, `solve-sdk-grid`).
- Other Apps (patterns seen in examples)
  - Latin Squares examples show `(solve "...string...")`: `CSP-Rules-Examples/LatinSquares/LatinSquares/9x9-#1-W5.clp`.
  - Pandiagonals show `(solve-list ...)`: `CSP-Rules-Examples/LatinSquares/Pandiagonals/13x13-DB#10-8-W4.clp`.
  - Map‑colouring examples show `(solve 4 30 ...)`: `CSP-Rules-Examples/Map-colouring/Tatham30-U22-tW4.clp` (first arg is number of colours; second is graph size in these examples).
  - Futoshiki examples show domain‑specific helpers like `(solve-tatham ...)`: `CSP-Rules-Examples/Futoshiki/Tatham/9x9-Recursive-W10.clp`.
- Tuning and Preferences (examples)
  - Toggle rule families and verbosity with `bind` forms; e.g., Subsets and Finned Fish toggles: `CSP-Rules-Examples/README.md`.
  - Whips/Braids and lengths: see `CSP-Rules-Examples/README.md`; controlled-bias study toggles under `CSP-Rules-Examples/Sudoku-b/cbg-000/launch.txt`.
  - `solve-w-preferences` is available generically and used in some Sudoku examples: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/MODULES/modules.clp`, `CSP-Rules-Examples/README.md`.
- Important Path Note (for later execution)
  - The default `?*CSP-Rules*` in `SudoRules-V20.1-config.clp` points to a maintainer’s path (e.g., `/Users/berthier/Projects/CSP-Rules/`). If running locally, set this to your absolute CSP‑Rules install path (or continue using your external install if you have one).

<a id="read-first"></a>
**What To Read First**
- Read in this order: Overview → Notation → Model → State → Trigger → Beyond → T&E
- Code orientation (optional, as needed):
  - CSP‑Rules readme and updates: `CSP-Rules/CSP-Rules-V2.1/README.md`, `UPDATES.md`.
  - Generic data contracts and knobs: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`, `.../globals.clp`.
  - Sudoku app structure and public API: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/*.clp` (especially `solve.clp`, `background.clp`, `templates.clp`).
  - Examples’ README for curated guidance by technique: `CSP-Rules-Examples/README.md`.

<a id="data"></a>
**Data & State Model**
- Core templates (generic)
  - `candidate` (status/label/context/flags): `CSP-Rules-Generic/GENERAL/templates.clp`.
  - `g-candidate` (grouped abstraction for g‑chains): `CSP-Rules-Generic/GENERAL/templates.clp`.
  - `csp-variable` and typed: `CSP-Rules-Generic/GENERAL/templates.clp`.
- Application extensions (Sudoku example)
  - App templates aligning Sudoku structure (numbers, rows, columns, blocks, squares): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp`.
  - Label system and typed link semantics with `labels-linked-by`, `labels-linked`: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp`.
- Link lifecycle
  - App defines semantics; generic asserts links/glinks in `init-links` with specific saliences: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`; related salience setup: `.../GENERAL/saliences.clp`.
- Runtime globals and counters
  - Generic knobs and counters: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp` (verbosity, family toggles, ORk, limits, T&E/DFS).
  - App parameters and toggles (Sudoku): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp`.
- Contradictions and context
  - Contradictions: no candidate left for a variable; contexts used by T&E/DFS (generic T&E files, candidate context slots).

<a id="public"></a>
**Public Surface Area**
- Load order
  - Per‑app config defines `?*CSP-Rules*`, computes directories, and loads generic/app `globals` before batching loaders: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp`.
  - Generic loader batches the app loader: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CSP-Rules-Generic-Loader.clp`.
- Entry points (Sudoku)
  - Strings/lists/grids: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp`.
  - File/batch: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve-files.clp`.
- Generic solve pattern
  - Size + list: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp`.
- Inputs/outputs and verbosity
  - Inputs: inline strings/lists/grids; file‑based sources; outputs: trace lines, stats with init/solve/total time (examples demonstrate formats).
  - Verbosity toggles: `?*print-*` globals in `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp` and Sudoku prints at `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp`.

<a id="config"></a>
**Configuration & Feature Flags**
- Paths and loaders
  - `?*CSP-Rules*` install path, dir separators, CLIPS version banner: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp`.
  - Computed dirs/loaders: `?*CSP-Rules-Generic-Loader*`, `?*Application-Loader*`.
- Rule selection and behavior (generic)
  - Families: `?*Subsets*`, `?*Bivalue-Chains*`, `?*z-Chains*`, `?*Whips*`, `?*Braids*`, typed/g‑variants (`?*G-Whips*`, `?*Typed-Whips*`, etc.) — see generic `globals.clp`.
  - ORk variants and ordering: `?*ORk-...*`, `?*ORk-Forcing-Whips-before-ORk-Whips*` — see generic `globals.clp`.
  - Length limits: `?*absolute-chains-max-length*` and family maxima; also `?*w-whips-max-length*`, `?*b-braids-max-length*` — see generic `globals.clp`.
  - Blocked vs unblocked behavior: `?*blocked-Subsets*`, `?*blocked-chains*`, `?*unblocked-behaviour*` — see generic `globals.clp`.
  - T&E/DFS: `?*TE1*`, `?*TE2*`, `?*TE3*`, `?*DFS*` — see generic `globals.clp`.
- App‑specific toggles (Sudoku)
  - Sub‑families and exotica: `?*FinnedFish*`, `?*Unique-Rectangles*`, `?*BUG*`, `?*Belt4*`, `?*J-Exocet*`, `?*Deadly-Patterns*`, Tridagons and guards: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp`.
  - Per‑technique print options: many `?*print-*` toggles under the same file.
- Examples for known settings
  - Subsets only: `CSP-Rules-Examples/README.md`.
  - Whips/Braids + T&E and lengths: `CSP-Rules-Examples/README.md`.
  - Controlled‑bias study presets: `CSP-Rules-Examples/Sudoku-b/cbg-000/launch.txt`.

<a id="resolution"></a>
**Resolution Theory & Rule System**
- BRT (Singles + ECP)
  - Generic ECP: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/ECP.clp`; generic Singles: `.../GENERAL/Single.clp`.
  - App NS/HS singles: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/NS.clp`, `.../HS.clp`.
- Chains and variants
  - Core families under `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-*` (COMMON, SPEED, MEMORY variants).
  - Exotic/ORk families under `.../CHAIN-RULES-EXOTIC/`.
  - Typed vs untyped and g‑variants controlled by globals (see Configuration).
- Ordering and salience
  - Salience definitions around init‑links and subsequent phases: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/saliences.clp`.
  - App‑specific saliences (e.g., Sudoku): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/saliences.clp`.
- Lifecycle
  - After config loads globals, Singles run, then `init-links` asserts links/glinks; chain/subset/uniqueness/exotic rules fire per salience until solved or limits reached.

<a id="naming"></a>
**Naming, Conventions, How To Extend**
Naming Patterns
- Techniques map to folders and rule names: `CHAIN-RULES-*` (whips, z‑chains, t‑whips, braids), app `SUBSETS/`, `UNIQUENESS/`, `EXOTIC/`.
- Trace lines mirror rule identifiers: e.g., `whip[n]`, `biv-chain`, `z-chain`, `hidden-single-in-a-row`.
- Sudoku typed variables/labels: `rc`, `rn`, `cn`, `bn`; core facts: `candidate`, `g-candidate`, `csp-variable` (see `CSP‑Rules‑Generic/GENERAL/templates.clp`).

CLIPS Conventions
- `deftemplate` for data, `defrule` with `salience` for sequencing, `deffunction` for utilities.
- Feature toggles via `defglobal` names `?*Name*` in generic/app globals.

Extension Recipe
- Define app templates extending generic ones under `YourApp/GENERAL/templates.clp`.
  - Implement `labels-linked-by`/`labels-linked` in `YourApp/GENERAL/background.clp` (cf. Sudoku: `SudoRules-V20.1/GENERAL/background.clp`).
- Reuse generic rules via the generic loader; add app‑specific rules in `GENERAL`, `SUBSETS`, `UNIQUENESS`, `EXOTIC`.
- Provide `YourApp-Vx.y-config.clp` and `YourApp-Loader.clp` like SudoRules.

<a id="comparison"></a>
**CSP‑Rules vs Traditional CSP Solvers**
Same
- Finite domains with candidates; constraint propagation over binary constraints (non‑binary encoded via extra variables).

Different
- Forward‑chaining production system (CLIPS + RETE) vs search‑first backtracking with AC/GAC.
- Pattern‑based resolution (chains, whips, braids) with a single reasoning stream per pattern.
- Systematic additional typed variables (e.g., `rn/cn/bn` in Sudoku) to expose structure.
- Optional T&E/DFS integrated under the same rule framework (`CSP-Rules-Generic/T&E+DFS/`).

<a id="glossary"></a>
**Glossary**
Candidate — A potential value for a variable with status/context slots (CSP‑Rules‑Generic/GENERAL/templates.clp).

g‑Candidate — Grouped candidate abstraction enabling g‑chains (CSP‑Rules‑Generic/GENERAL/templates.clp).

csp‑Variable — Structural variable facts representing modeled entities, including typed forms (CSP‑Rules‑Generic/GENERAL/templates.clp).

Label — Encoded identifier of a variable/value position; apps define typed labels (e.g., Sudoku’s `rc/rn/cn/bn`).

links/glinks — Logical adjacency between labels; g‑variants for grouped forms; computed via app `labels-linked*` and asserted by generic `init-links` (Sudo background: `SudoRules-V20.1/GENERAL/background.clp`; generic init: `CSP-Rules-Generic/GENERAL/init-links.clp`).

BRT — Basic Resolution Theory (Singles + ECP), generic rules for early eliminations (ECP: `CSP-Rules-Generic/GENERAL/ECP.clp`; Singles: `.../GENERAL/Single.clp`; app NS/HS: `SudoRules-V20.1/GENERAL/NS.clp`, `HS.clp`).

Chains — Technique families: bivalue, z‑chains, t‑whips, whips, g‑whips, braids; ORk exotic variants in `CSP-Rules-Generic/CHAIN-RULES-EXOTIC/`.

T&E(n) — Trial & Error to depth n; DFS — depth‑first search augmentations in `CSP-Rules-Generic/T&E+DFS/`.

Salience — Rule priority controlling firing order; configured in generic/app `.../saliences.clp`.

Density — Measure printed in stats reflecting links/candidates after `init-links`.

Blocked/Unblocked — Controlled interruption vs free running of rule families via toggles (`?*blocked-Subsets*`, `?*blocked-chains*`, etc. in `CSP-Rules-Generic/GENERAL/globals.clp`).

Preferences — Alternative technique selection via `solve-w-preferences` (CSP‑Rules‑Generic/MODULES/modules.clp).

Rating Types — Derived from generic and app settings (`CSP-Rules-Generic/GENERAL/globals.clp`).

**See Also**
- Model: conceptual API and objects — [Model](Model.md)
- Graphs: nodes/edges, build, pruning — [Graphs](Graphs.md)
- State: facts + globals and lifecycle — [State](State.md)
- Pattern taxonomy: Scope → Trigger → Action — [Trigger](Trigger.md)
- Notation and real traces — [Notation](Notation.md)
- Non‑binary → binary modeling — [Beyond](Beyond.md)
- Trial & Error and DFS — [T&E](T&E.md)

<a id="schema"></a>
**Schema & Flow**
```
Inputs                          Load & Setup                        Core Facts & Links                 Solve Phases                     Effects / Output
------                          -------------                       ------------------                 -----------                      ----------------
String/List/Grid/File  ->  Config (`*-config.clp`)  ->  Generic Loader  ->  App Loader  ->  Initial Facts
                            - sets `?*CSP-Rules*`        (`CSP-Rules-Generic-Loader.clp`)   (csp-variable, candidate, g-candidate)
                            - computes dirs              batches app loader                 + Labels per app (e.g., rc/rn/cn/bn)
                            - loads generic/app globals  (`SudoRules-Loader.clp` etc.)

Initial Facts  ->  init-links (generic)  ->  Links/Glinks asserted  ->  Density/Counts
                   `CSP-Rules-Generic/GENERAL/init-links.clp`        (based on app `labels-linked*`)    (printed if enabled)

Then:
  BRT: Singles + ECP        Chains/Subsets/Uniqueness/Exotic          Optional T&E/DFS
  - `ECP.clp`, NS/HS        - `CHAIN-RULES-*` (+ app dirs)            - `T&E+DFS/*`
  - app `NS.clp`, `HS.clp`

Loop until solved / contradiction / limits:
  - Rule firings eliminate candidates or assert values (c-values)
  - Stats/trace printed per `?*print-*` globals
```

Key refs: generic templates `CSP-Rules-Generic/GENERAL/templates.clp`, Sudoku background `SudoRules-V20.1/GENERAL/background.clp`, init-links `CSP-Rules-Generic/GENERAL/init-links.clp`, saliences `.../GENERAL/saliences.clp`.

<a id="fast"></a>
**Fast vs Thorough Config**
- Fast (quick feedback; minimal logging)
  - Use SPEED chains (default): see `CSP-Rules-Generic/GENERAL/globals.clp` (`?*chain-rules-optimisation-type* = SPEED`).
  - Suppress prints:
    - `(bind ?*print-actions* FALSE)` `(bind ?*print-levels* FALSE)` `(bind ?*print-solution* FALSE)`
    - `(bind ?*print-RS-after-Singles* FALSE)` `(bind ?*print-final-RS* FALSE)`
    - Example presets: `CSP-Rules-Examples/Sudoku-b/cbg-000/launch.txt`.
  - Keep rule families tight:
    - Start with BRT only (defaults: most families are FALSE in generic `globals.clp`).
    - If needed, enable short Whips only: `(bind ?*Whips* TRUE)` `(bind ?*whips-max-length* 3)` or `5`.
    - Avoid typed/g/ORk variants and T&E/DFS.

- Thorough (hard instances; deeper reasoning)
  - Still SPEED by default; consider MEMORY if RAM is constrained:
    - In app config, uncomment: `(bind ?*chain-rules-optimisation-type* MEMORY)` (see `SudoRules-V20.1-config.clp`).
  - Enable broader families and lengths (pick per domain):
    - `(bind ?*Whips* TRUE)` `(bind ?*whips-max-length* 7)` to `12`
    - `(bind ?*Braids* TRUE)` `(bind ?*braids-max-length* 7)` to `12`
    - Optionally typed/g-variants: `(bind ?*Typed-Whips* TRUE)`, `(bind ?*G-Whips* TRUE)` and respective max lengths.
    - App‑specific patterns for Sudoku: `(bind ?*Subsets* TRUE)` `(bind ?*FinnedFish* TRUE)` `(bind ?*Unique-Rectangles* TRUE)` `(bind ?*Deadly-Patterns* TRUE)` (see `SudoRules-V20.1/GENERAL/globals.clp`).
    - ORk families sparingly (heavy): e.g., `(bind ?*OR2-Whips* TRUE)` with modest `?*OR2-whips-max-length*` (see generic globals and Sudoku modules `TRID-ORk-*-module.clp`).
  - Consider limited T&E for classification or last resort:
    - `(bind ?*TE1* TRUE)` and moderate chain limits; examples in `CSP-Rules-Examples/README.md`.
  - Reduce prints but keep key summaries:
    - `(bind ?*print-actions* FALSE)` `(bind ?*print-solution* TRUE)` `(bind ?*print-final-RS* TRUE)`.

Notes
- Use examples to mirror technique‑specific presets (e.g., Subsets‑only, Tridagons, controlled‑bias cbg settings).
- Watchers like `(watch rules/facts)` are useful for ad‑hoc investigation but keep them off for performance.




Your solver already uses the right building blocks (boolean candidate tensor, batch dimension, vectorized reductions, slices). Extend that to chains by treating the chain frontier as a batch of “parallel contexts,” and to T&E by treating hypotheses as a batch
  over candidate masks.
  - You don’t need matmul/einsum or bitsets if you prefer booleans; broadcasting + axis‑wise sums will get you there.