**TL;DR**
- This workspace contains a local copy of the CSP-Rules codebase and the companion examples repository: `CSP-Rules/CSP-Rules-V2.1/` and `CSP-Rules-Examples/`.
- CSP-Rules is a pattern-based (rule-based) solver for finite binary CSPs implemented in CLIPS; an application layer (e.g., Sudoku) extends generic templates and rules.
- This document summarizes architecture (generic core → app modules), shows where to look for data/state models and loaders, and captures usage patterns directly from the examples (no execution required).

**Architecture At A Glance**
- Generic Core
  - Loader orchestrates generic modules and then batches the active application loader: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CSP-Rules-Generic-Loader.clp:1073`.
  - Templates (data contracts), runtime globals, solve loop and salience scheduling: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`, `.../globals.clp`, `.../solve.clp:125`, `.../saliences.clp`.
  - Link initialization and play loop: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp`, `.../play.clp`.
  - Chains and variants (common/exotic; speed/memory): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-*/*`.
  - T&E + DFS augmentations: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/T&E+DFS/`.
- Applications (example: Sudoku)
  - Per‑app loader and modules: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/SudoRules-Loader.clp` (batched by the generic loader), `.../GENERAL/`, `.../SUBSETS/`, `.../UNIQUENESS/`, `.../EXOTIC/`.
  - App templates and background (labels, typed variables, link semantics): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp`, `.../background.clp`.
  - App solve API (selected entries): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp:500` (`solve`), `:469` (`solve-sudoku-string`), `:861` (`solve-sudoku-list`), `:1184` (`solve-sudoku-grid`).
  - Batch/file helpers: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve-files.clp:63` and friends.
- Configuration
  - Per‑app config defines install path and loaders and can auto-batch the generic loader: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp:49`, `:87`, `:90`, `:890`.

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
  - `overview.md` — This analysis and guide. `README.md` — Workspace readme.

**Examples‑First Usage Cheat‑Sheet (No‑Run)**
- Load Order Convention (from app config and loaders)
  - In CLIPS REPL: load a per‑app config (it sets `?*CSP-Rules*` to your install and defines loader paths), then the config batches the generic loader which in turn batches the app loader.
  - Key config/globals and loader symbols in Sudoku config: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp:49` (`?*CSP-Rules*`), `:87` (`?*CSP-Rules-Generic-Loader*`), `:90` (`?*Application-Loader*`), and auto‑batch on selection `:890`.
- Typical Sudoku Invocations (from source)
  - String/grid based: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp:500` (`(solve "...puzzle...")`), `:469` (`solve-sudoku-string`), `:1184` (`solve-sudoku-grid ...)`).
  - File/batch helpers: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve-files.clp:63` (`solve-grid-from-text-file ...`), `:99` (`solve-nth-grid-from-text-file ...`), `:111` (`solve-n-grids-after-first-p-from-text-file ...)`, `:430` (`solve-sdk-grid ...)`.
- Other Apps (patterns seen in examples)
  - Latin Squares examples show `(solve "...string...")`: `CSP-Rules-Examples/LatinSquares/LatinSquares/9x9-#1-W5.clp:12`.
  - Pandiagonals show `(solve-list ...)`: `CSP-Rules-Examples/LatinSquares/Pandiagonals/13x13-DB#10-8-W4.clp:25`.
  - Map‑colouring examples show `(solve 4 30 ...)`: `CSP-Rules-Examples/Map-colouring/Tatham30-U22-tW4.clp:3` (first arg is colors; second is graph size in these examples).
  - Futoshiki examples show domain‑specific helpers like `(solve-tatham ...)`: `CSP-Rules-Examples/Futoshiki/Tatham/9x9-Recursive-W10.clp:3`.
- Tuning and Preferences (examples)
  - Toggle rule families and verbosity with `bind` forms; e.g., Subsets and Finned Fish toggles: `CSP-Rules-Examples/README.md:62`, `:64`.
  - Whips/Braids and lengths: `CSP-Rules-Examples/README.md:168`–`:181`; controlled-bias study toggles under `CSP-Rules-Examples/Sudoku-b/cbg-000/launch.txt:23`–`:65`.
  - `solve-w-preferences` is available generically and used in some Sudoku examples: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/MODULES/modules.clp:98`, `CSP-Rules-Examples/README.md:99`, `:156`.
- Important Path Note (for later execution)
  - The Sudoku config currently points `?*CSP-Rules*` to `/home/jake/Developer/CSP-Rules/` (`CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp:49`). If running within this workspace, either keep using your external install or update this path to this workspace’s `CSP-Rules/` absolute path.

**What To Read First**
- Core concept overview and supported apps: `CSP-Rules/CSP-Rules-V2.1/README.md`.
- Change history and features: `CSP-Rules/CSP-Rules-V2.1/UPDATES.md`.
- Generic data contracts and runtime knobs: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/templates.clp`, `.../globals.clp`.
- Sudoku app structure and public API: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/*.clp` (especially `solve.clp`, `background.clp`, `templates.clp`).
- Examples’ README for curated guidance by technique: `CSP-Rules-Examples/README.md`.

**Data & State Model**
- Core templates (generic)
  - `candidate` (status/label/context/flags): `CSP-Rules-Generic/GENERAL/templates.clp:79`.
  - `g-candidate` (grouped abstraction for g‑chains): `CSP-Rules-Generic/GENERAL/templates.clp:101`.
  - `csp-variable` and typed: `CSP-Rules-Generic/GENERAL/templates.clp:123`.
- Application extensions (Sudoku example)
  - App templates aligning Sudoku structure (numbers, rows, columns, blocks, squares): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/templates.clp:34`.
  - Label system and typed link semantics with `labels-linked-by`, `labels-linked`: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/background.clp:407`, :430.
- Link lifecycle
  - App defines semantics; generic asserts links/glinks in `init-links` with specific saliences: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/init-links.clp:49`; related salience setup: `.../GENERAL/saliences.clp:448`.
- Runtime globals and counters
  - Generic knobs and counters: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp` (verbosity at :1801+; chain toggles :448+; ORk :485+; limits :562+; T&E/DFS :753+).
  - App parameters and toggles (Sudoku): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp:44`, :113, :119–:131, :156+, :286+.
- Contradictions and context
  - Contradictions: no candidate left for a variable; contexts used by T&E/DFS (generic T&E files, candidate context slots).

**Public Surface Area**
- Load order
  - Per‑app config defines `?*CSP-Rules*`, computes directories, and loads generic/app `globals` before batching loaders: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp:49`, :87, :90, :890.
  - Generic loader batches the app loader: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CSP-Rules-Generic-Loader.clp:1073`.
- Entry points (Sudoku)
  - Strings/lists/grids: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve.clp:500` (`solve`), :469 (`solve-sudoku-string`), :861 (`solve-sudoku-list`), :1184 (`solve-sudoku-grid`).
  - File/batch: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/solve-files.clp:63` (`solve-grid-from-text-file`), :99 (`solve-nth-grid-from-text-file`), :111 (batch after p), :430 (`solve-sdk-grid`).
- Generic solve pattern
  - Size + list: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/solve.clp:125`.
- Inputs/outputs and verbosity
  - Inputs: inline strings/lists/grids; file‑based sources; outputs: trace lines, stats with init/solve/total time (examples demonstrate formats).
  - Verbosity toggles: `?*print-*` globals in `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp:1801+` and Sudoku prints at `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp:653+`.

**Configuration & Feature Flags**
- Paths and loaders
  - `?*CSP-Rules*` install path, dir separators, CLIPS version banner: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1-config.clp:49`, :31, :65.
  - Computed dirs/loaders: `?*CSP-Rules-Generic-Loader*`, `?*Application-Loader*`: `:87`, `:90`.
- Rule selection and behavior (generic)
  - Families: `?*Subsets*`, `?*Bivalue-Chains*`, `?*z-Chains*`, `?*Whips*`, `?*Braids*`, typed/g‑variants (`?*G-Whips*`, `?*Typed-Whips*`, etc.): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp:399`, :448–:473, :457–:462, :468–:473, :470–:471.
  - ORk variants and ordering: `?*ORk-...*`, `?*ORk-Forcing-Whips-before-ORk-Whips*`: `.../globals.clp:485`–:540, :543.
  - Length limits: `?*absolute-chains-max-length*` and family maxima: `.../globals.clp:562`–:579; also `?*w-whips-max-length*`, `?*b-braids-max-length*` at :737–:738.
  - Blocked vs unblocked behavior: `?*blocked-Subsets*`, `?*blocked-chains*`, `?*unblocked-behaviour*`: `.../globals.clp:378`–:387.
  - T&E/DFS: `?*TE1*`, `?*TE2*`, `?*TE3*`, `?*DFS*`: `.../globals.clp:753`–:775.
- App‑specific toggles (Sudoku)
  - Sub‑families and exotica: `?*FinnedFish*`, `?*Unique-Rectangles*`, `?*BUG*`, `?*Belt4*`, `?*J-Exocet*`, `?*Deadly-Patterns*`, Tridagons and guards: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/globals.clp:113`, :119–:131, :156–:163, :283–:301.
  - Per‑technique print options: many `?*print-*` toggles under `:653+`.
- Examples for known settings
  - Subsets only: `CSP-Rules-Examples/README.md:62`, :64.
  - Whips/Braids + T&E and lengths: `CSP-Rules-Examples/README.md:168`–:181.
  - Controlled‑bias study presets: `CSP-Rules-Examples/Sudoku-b/cbg-000/launch.txt:23`–:65.

**Resolution Theory & Rule System**
- BRT (Singles + ECP)
  - Generic ECP: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/ECP.clp`; generic Singles: `.../GENERAL/Single.clp`.
  - App NS/HS singles: `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/NS.clp`, `.../HS.clp`.
- Chains and variants
  - Core families under `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/CHAIN-RULES-*` (COMMON, SPEED, MEMORY variants).
  - Exotic/ORk families under `.../CHAIN-RULES-EXOTIC/`.
  - Typed vs untyped and g‑variants controlled by globals (see Configuration).
- Ordering and salience
  - Salience definitions around init‑links and subsequent phases: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/saliences.clp:448`, :515–:524.
  - App‑specific saliences (e.g., Sudoku): `CSP-Rules/CSP-Rules-V2.1/SudoRules-V20.1/GENERAL/saliences.clp:1`.
- Lifecycle
  - After config loads globals, Singles run, then `init-links` asserts links/glinks; chain/subset/uniqueness/exotic rules fire per salience until solved or limits reached.

**Naming, Conventions, How To Extend**
Naming Patterns
- Techniques map to folders and rule names: `CHAIN-RULES-*` (whips, z‑chains, t‑whips, braids), app `SUBSETS/`, `UNIQUENESS/`, `EXOTIC/`.
- Trace lines mirror rule identifiers: e.g., `whip[n]`, `biv-chain`, `z-chain`, `hidden-single-in-a-row`.
- Sudoku typed variables/labels: `rc`, `rn`, `cn`, `bn`; core facts: `candidate`, `g-candidate`, `csp-variable` (CSP‑Rules‑Generic/GENERAL/templates.clp:79, :101, :123).

CLIPS Conventions
- `deftemplate` for data, `defrule` with `salience` for sequencing, `deffunction` for utilities.
- Feature toggles via `defglobal` names `?*Name*` in generic/app globals.

Extension Recipe
- Define app templates extending generic ones under `YourApp/GENERAL/templates.clp`.
- Implement `labels-linked-by`/`labels-linked` in `YourApp/GENERAL/background.clp` (cf. Sudoku: `SudoRules-V20.1/GENERAL/background.clp:407`, :430).
- Reuse generic rules via the generic loader; add app‑specific rules in `GENERAL`, `SUBSETS`, `UNIQUENESS`, `EXOTIC`.
- Provide `YourApp-Vx.y-config.clp` and `YourApp-Loader.clp` like SudoRules.

**CSP‑Rules vs Traditional CSP Solvers**
Same
- Finite domains with candidates; constraint propagation over binary constraints (non‑binary encoded via extra variables).

Different
- Forward‑chaining production system (CLIPS + RETE) vs search‑first backtracking with AC/GAC.
- Pattern‑based resolution (chains, whips, braids) with a single reasoning stream per pattern.
- Systematic additional typed variables (e.g., `rn/cn/bn` in Sudoku) to expose structure.
- Optional T&E/DFS integrated under the same rule framework (`CSP-Rules-Generic/T&E+DFS/`).

**Glossary**
Candidate — A potential value for a variable with status/context slots (CSP‑Rules‑Generic/GENERAL/templates.clp:79).

g‑Candidate — Grouped candidate abstraction enabling g‑chains (CSP‑Rules‑Generic/GENERAL/templates.clp:101).

csp‑Variable — Structural variable facts representing modeled entities, including typed forms (CSP‑Rules‑Generic/GENERAL/templates.clp:123).

Label — Encoded identifier of a variable/value position; apps define typed labels (e.g., Sudoku’s `rc/rn/cn/bn`).

links/glinks — Logical adjacency between labels; g‑variants for grouped forms; computed via app `labels-linked*` and asserted by generic `init-links` (Sudo background: `SudoRules-V20.1/GENERAL/background.clp:430`; generic init: `CSP-Rules-Generic/GENERAL/init-links.clp:49`).

BRT — Basic Resolution Theory (Singles + ECP), generic rules for early eliminations (ECP: `CSP-Rules-Generic/GENERAL/ECP.clp`; Singles: `.../GENERAL/Single.clp`; app NS/HS: `SudoRules-V20.1/GENERAL/NS.clp`, `HS.clp`).

Chains — Technique families: bivalue, z‑chains, t‑whips, whips, g‑whips, braids; ORk exotic variants in `CSP-Rules-Generic/CHAIN-RULES-EXOTIC/`.

T&E(n) — Trial & Error to depth n; DFS — depth‑first search augmentations in `CSP-Rules-Generic/T&E+DFS/`.

Salience — Rule priority controlling firing order; configured in generic/app `.../saliences.clp`.

Density — Measure printed in stats reflecting links/candidates after `init-links`.

Blocked/Unblocked — Controlled interruption vs free running of rule families via toggles (`?*blocked-Subsets*`, `?*blocked-chains*`, etc. in `CSP-Rules-Generic/GENERAL/globals.clp:378`–:387).

Preferences — Alternative technique selection via `solve-w-preferences` (CSP‑Rules‑Generic/MODULES/modules.clp:98).

Rating Types — Derived from generic and app settings (`CSP-Rules-Generic/GENERAL/globals.clp:1510`–:1512).

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

Key refs: generic templates `CSP-Rules-Generic/GENERAL/templates.clp`, Sudoku background `SudoRules-V20.1/GENERAL/background.clp:407,430`, init-links `CSP-Rules-Generic/GENERAL/init-links.clp:49`, saliences `.../GENERAL/saliences.clp:448`.

**Fast vs Thorough Config**
- Fast (quick feedback; minimal logging)
  - Use SPEED chains (default): `CSP-Rules-Generic/GENERAL/globals.clp:360` (`?*chain-rules-optimisation-type* = SPEED`).
  - Suppress prints:
    - `(bind ?*print-actions* FALSE)` `(bind ?*print-levels* FALSE)` `(bind ?*print-solution* FALSE)`
    - `(bind ?*print-RS-after-Singles* FALSE)` `(bind ?*print-final-RS* FALSE)`
    - Example presets: `CSP-Rules-Examples/Sudoku-b/cbg-000/launch.txt:23`–:29.
  - Keep rule families tight:
    - Start with BRT only (defaults: most families are FALSE in `CSP-Rules-Generic/GENERAL/globals.clp:399`, :448+).
    - If needed, enable short Whips only: `(bind ?*Whips* TRUE)` `(bind ?*whips-max-length* 3)` or `5`.
    - Avoid typed/g/ORk variants and T&E/DFS.

- Thorough (hard instances; deeper reasoning)
  - Still SPEED by default; consider MEMORY if RAM is constrained:
    - In app config, uncomment: `(bind ?*chain-rules-optimisation-type* MEMORY)` e.g., `SudoRules-V20.1-config.clp:146` (commented example).
  - Enable broader families and lengths (pick per domain):
    - `(bind ?*Whips* TRUE)` `(bind ?*whips-max-length* 7)` to `12`
    - `(bind ?*Braids* TRUE)` `(bind ?*braids-max-length* 7)` to `12`
    - Optionally typed/g-variants: `(bind ?*Typed-Whips* TRUE)`, `(bind ?*G-Whips* TRUE)` and respective max lengths.
    - App‑specific patterns for Sudoku: `(bind ?*Subsets* TRUE)` `(bind ?*FinnedFish* TRUE)` `(bind ?*Unique-Rectangles* TRUE)` `(bind ?*Deadly-Patterns* TRUE)` (see `SudoRules-V20.1/GENERAL/globals.clp:113`, :119–:131, :156+).
    - ORk families sparingly (heavy): e.g., `(bind ?*OR2-Whips* TRUE)` with modest `?*OR2-whips-max-length*` (see generic globals around :485+ and Sudoku modules `TRID-ORk-*-module.clp`).
  - Consider limited T&E for classification or last resort:
    - `(bind ?*TE1* TRUE)` and moderate chain limits; examples at `CSP-Rules-Examples/README.md:168`–:181`.
  - Reduce prints but keep key summaries:
    - `(bind ?*print-actions* FALSE)` `(bind ?*print-solution* TRUE)` `(bind ?*print-final-RS* TRUE)`.

Notes
- Use examples to mirror technique‑specific presets (e.g., Subsets‑only, Tridagons, controlled‑bias cbg settings).
- Watchers like `(watch rules/facts)` are useful for ad‑hoc investigation but keep them off for performance.
