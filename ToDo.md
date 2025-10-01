# Docs Cleanup and Flow To‑Dos

This checklist organizes the documentation fixes and improvements we discussed. We’ll work through these items in order, validating references as we go.

## 1) Fix Broken Links and Path Notes (P0)
- [x] README: resolve `Docs/Itinerary.md` link
  - [ ] Option A: add `Docs/Itinerary.md` placeholder
  - [x] Option B: remove the link from README
- [x] Docs/Overview.md: correct file location mention of the guide
  - [x] Replace “root `overview.md`” with `Docs/Overview.md`
- [x] Docs/Overview.md: correct `?*CSP-Rules*` default path note
  - [x] Reflect default value in `SudoRules-V20.1-config.clp:49` (currently `/Users/berthier/...`)
  - [x] Advise users to set an absolute local path (or rely on external install)

## 2) Reorder Onboarding Flow (P0)
- [x] README “Docs — Start Here” order → `Overview → Notation → Model → State → Trigger → Beyond → T&E`
- [x] Docs/Overview.md: move/read “What To Read First” under TL;DR and emphasize Notation early
- [x] Add one‑line rationale in README and Overview for the order change

## 3) Stabilize Pointers and Reduce Brittleness (P0)
- [x] Replace most `file:line` references with “file + function/rule/template name” across docs (Model, Trigger, Notation, Beyond, T&E)
- [x] Keep a few verified anchors where line numbers add clear value; state this policy at the top of each affected doc
- [x] Add a “Code Index” or “References” section per doc to park detailed pointers (each doc includes a references/index section)

## 4) Correct Specific References in State.md (P0)
- [x] Update incorrect line numbers for generic templates
- [x] Prefer “file + template name” where possible to avoid drift

## 5) Notation Improvements (P1)
- [x] Move “End‑to‑End Excerpts” near the top; cross‑link from README/Overview
- [x] Confirm helper file references (existence and names):
  - [x] `CSP-Rules-Generic/GENERAL/generic-output.clp`
  - [x] `CSP-Rules-Generic/GENERAL/parameters.clp`
  - [x] `SudoRules-V20.1/GENERAL/nrc-output.clp`
- [x] Add a brief “How to read a chain line in 30 seconds” callout

## 6) Editorial Prefaces and Clarity (P1)
- [x] Add “Reference note” prefaces to: Overview, Model, State, Trigger, T&E, Notation, Beyond
- [x] Trim early dense pointer sections; move deep details to the end under “References/Index”
- [x] Add cross‑links among docs for smooth navigation (via See Also sections)

## 7) Cross‑Reference Audit (P1)
- [x] Grep all docs for backticked `file:line` style references; fix or convert
- [x] Spot‑check app‑specific examples (Map, Kakuro, Latin, Slitherlink, Futoshiki, Hidato, Numbrix) to ensure claimed files exist

## 8) Optional Additions (P2)
- [ ] Add `Docs/Itinerary.md` as a simple roadmap template (if we keep the link)
- [x] Add a short “Doc maintenance” note explaining the pointer policy (added in each doc as a reference note)

## 9) Execution Order (Tracking)
- [x] Apply README.md updates
- [x] Apply Docs/Overview.md updates
- [x] Apply Docs/State.md corrections
- [x] Apply Docs/Notation.md edits
- [x] Sweep pointer conversions across Model/Trigger/Beyond/T&E
- [x] Final sanity pass: validate links and file references
