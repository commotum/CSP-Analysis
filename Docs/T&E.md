## Contents
- [What T&E Is](#what)
- [Flags & Entry Points](#flags)
- [Data Structures](#data)
- [TE Flow](#flow)
- [DFS](#dfs)
- [Forcing T&E](#forcing)
- [Integrated Design](#integrated)
- [Single Stream vs Search](#stream)
- [Used vs Not Used](#used)
- [Pointers](#pointers)
- [See Also](#see)

<a id="what"></a>
**Trial & Error (T&E) and DFS in CSP‑Rules**

This notes how T&E works in CSP‑Rules: what it does, the data it uses, and how it stays integrated with the same rule engine (no separate solver).

**What T&E Is (Plain)**
- Hypothesize a candidate as true in a child “context”, run the normal rules, and see if that leads to a contradiction. If it does, the hypothesis is impossible, so the candidate is eliminated in the parent. If not, discard the temporary context and try the next hypothesis.
- Depth 1 tries one hypothesis at a time; higher depths nest hypotheses. DFS is the same idea but pursues a search branch until a solution or contradiction.

<a id="flags"></a>
**Flags & Entry Points**
- Toggles (generic):
  - `?*TE1*`, `?*TE2*`, `?*TE3*` — enable T&E at depth 1/2/3.
  - `?*Forcing-TE*` — enable forcing variants using T&E on pairs.
  - `?*DFS*` — enable depth‑first search.
  - `?*restrict-TE-targets*` — optional restriction on which candidates can be hypothesized.
  - Printing: `?*print-hypothesis*`, `?*print-phase*`, `?*print-actions*` (and more in `.../GENERAL/globals.clp:1801+`).
  Pointers: `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/GENERAL/globals.clp:753`–`:775` (T&E/DFS) and `:1801+` (prints).

<a id="data"></a>
**Core Data Structures (Facts & Globals)**
- Facts (shared with all rules):
  - `candidate` — with `status` `cand` or `c-value` (decided). Duplicated into child contexts. E.g., TE1 init: `T&E+DFS/T&E1.clp:23`, `:36`.
  - `csp-variable` and `is-csp-variable-for-label` — used to detect contradictions: if a variable has no remaining candidate in a child, the hypothesis is refuted. See `T&E+DFS/T&E1.clp:86`.
  - Link facts: `csp-linked` and `exists-link` — re‑computed inside each child context by the normal init‑links phase (not copied). Cleaned on context teardown. See `.../GENERAL/init-links.clp` and TE clean rules: `T&E+DFS/T&E1.clp:56`–`:83`.
  - Orchestration facts: `context (name, parent, depth, generating-cand)`, `technique`, `phase`, `phase-productive-in-context`, `clean-and-retract`. These steer hypothesis creation, iteration, and cleanup. See `T&E+DFS/T&E1.clp:129`–`:179`.
- Globals (selected):
  - Counters: `?*context-counter*`, `?*nb-candidates*`, `?*nb-csp-variables-solved*`, `?*DFS-max-depth*` (DFS only). E.g., `T&E1.clp:152`, `DFS.clp:243`.
  - Caches: `?*links*`, `?*glinks*` store global link pairs used by generic “linked” helpers. These are maintained for context 0 only (init‑links calls `add-link` only in context 0) and are not duplicated per T&E context. See `CSP-Rules-Generic/GENERAL/init-links.clp:101`, `:121` and `.../generic-background.clp:230`, `:273`.

How each is used by T&E
- candidates/c-values — copied from parent to child (c‑values copied with `flag 0`). The child starts by asserting the “generating” value (hypothesis) as a `c-value`. `T&E1.clp:164`.
- csp-variables/mapping — unchanged; used to test contradiction: “there exists a csp-variable with no remaining candidate” in the child ⇒ eliminate the hypothesis in the parent. `T&E1.clp:86`–`:105`.
- links — not copied; the child asserts `technique ?cont BRT` so BRT and the usual init‑links rules recompute `csp-linked`/`exists-link` in the child context. Clean rules retract these when done. `T&E1.clp:158`, clean: `:56`–`:83`.
- globals — counters/printing updated as usual; link caches and density are only for context 0 and are not relied upon to decide contradictions in children (children use their own link facts + app “labels‑linked” functions).

<a id="flow"></a>
**TE(1), TE(2), TE(3) Flow**
- Start (depth 1):
  - After BRT in context 0, assert `(technique 0 TE)` and `(phase 0 1)`. `T&E1.clp:120`–`:130`.
  - Generate a child context for an untried candidate: `(context (name new) (parent 0) (depth 1) (generating-cand L))`. Duplicate parent facts, assert the hypothesis as `c-value`, and run rules. `T&E1.clp:140`–`:172`.
  - If a contradiction is detected in the child, retract the generating candidate in the parent, mark the phase productive, and clean the child. `T&E1.clp:86`–`:107`.
  - If no contradiction is found after rules fire, just clean the child (`NO CONTRADICTION FOUND...`) and move on. `T&E1.clp:111`–`:118`.
  - Iterate phases if the last phase was productive (to try all candidates again under the new parent state). `T&E1.clp:174`–`:189`.
- Depth 2/3: same pattern, but contexts can be generated at `depth = 1` (children of 0) and at `depth = 2` (children of children). See `T&E2.clp` and `T&E3.clp` for mirrored rules and per‑depth phase control.

<a id="dfs"></a>
**DFS (Depth‑First Search)**
- Uses the same context mechanism and normal rules; differs by pursuing hypotheses until either a solution is detected in a context or a contradiction occurs; tracks `?*DFS-max-depth*`. See `T&E+DFS/DFS.clp:201`–`:252`.
- Solution detection rule in DFS: if every `csp-variable` has exactly one `c-value` present in the context, assert `solution-found`. `DFS.clp:150`–`:173`.

<a id="forcing"></a>
**Forcing T&E (Pairs)**
- For some strategies, T&E is run on both choices of a bivalue (or a candidate pair), then the two derived contexts are compared:
  - If both branches delete the same candidate, delete it in the parent (a “common deletion”).
  - If both branches assert the same `c-value`, assert it in the parent (a “common assertion”).
- Implemented with compare helpers and result templates like `F2TE-bivalue-pair-simple-result`; see `T&E+DFS/Forcing2-TE.clp:1`–`:120`.

<a id="integrated"></a>
**Important: Integrated, Not a Separate Solver**
- T&E/DFS don’t replace the rule engine; they schedule new contexts and seed them with a hypothesis, then let the same BRT/links/chains rules run inside that context.
- All eliminations and assertions still come from rules (Singles, ECP, chains, subsets, etc.) — not from ad‑hoc procedural code.
- Cleanup retracts the child’s facts so working memory stays bounded.

<a id="stream"></a>
**“Single Stream of Reasoning” vs. Search**
- Pattern rules (e.g., whips, braids) express a single chain of implications inside one rule firing — they don’t branch the state. Even ORk variants are encoded as specific rule families that match structured disjunctions within a single derivation (no recursive search).
- T&E/DFS is different: it explicitly branches the global state by asserting a hypothesis in a new context; propagation happens independently in that branch. If the branch refutes itself, the parent learns “not that hypothesis.”
- Practically: enable chains to get human‑style, linear deductions; enable T&E/DFS when you accept proof by contradiction or need completeness.

<a id="used"></a>
**What Is/Isn’t Used By T&E**
- Used directly:
  - `candidate`/`c-value`, `csp-variable`/`is-csp-variable-for-label`, `context`/`technique`/`phase`, link facts (`csp-linked`/`exists-link`) inside the child, and any rule families you enabled.
- Not copied; recomputed per child:
  - Links/glinks and any chain/typed‑chain facts (declared per context by the usual init and chain rules).
- Not relied upon inside child contexts:
  - Global link caches and density (they are maintained for context 0 only).

<a id="pointers"></a>
**Pointers**
- TE(1): `CSP-Rules/CSP-Rules-V2.1/CSP-Rules-Generic/T&E+DFS/T&E1.clp` (init, contradiction/no‑contradiction, generate/iterate, clean).
- TE(2)/TE(3): `.../T&E2.clp`, `.../T&E3.clp` (mirrored rules per depth).
- DFS: `.../DFS.clp` (generate children, detect solution/contrad, clean).
- Forcing T&E: `.../Forcing2-TE.clp` (pair compare), `.../ORk-Forcing-TE.clp`.
- Flags and prints: `CSP-Rules-Generic/GENERAL/globals.clp` (T&E/DFS toggles at ~:753–:775; prints at :1801+).

<a id="see"></a>
**See Also**
- State: how facts/globals are copied or recomputed in child contexts — [State](State.md)
- Model: object vocabulary for reading T&E rules — [Model](Model.md)
- Overview: load order and where T&E is hooked — [Overview](Overview.md)
- Trigger: pattern families that run inside T&E contexts — [Trigger](Trigger.md)
- Notation: reading T&E/DFS prints and implications — [Notation](Notation.md)
- Beyond: contradictions over csp‑variables — [Beyond](Beyond.md)
