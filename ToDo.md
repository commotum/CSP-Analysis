# Docs Cleanup and Flow To‑Dos

This checklist organizes the documentation fixes and improvements we discussed. We’ll work through these items in order, validating references as we go.

## 1) Fix Broken Links and Path Notes (P0)
- [ ] README: resolve `Docs/Itinerary.md` link
  - [ ] Option A: add `Docs/Itinerary.md` placeholder
  - [ ] Option B: remove the link from README
- [ ] Docs/Overview.md: correct file location mention of the guide
  - [ ] Replace “root `overview.md`” with `Docs/Overview.md`
- [ ] Docs/Overview.md: correct `?*CSP-Rules*` default path note
  - [ ] Reflect default value in `SudoRules-V20.1-config.clp:49` (currently `/Users/berthier/...`)
  - [ ] Advise users to set an absolute local path (or rely on external install)

## 2) Reorder Onboarding Flow (P0)
- [ ] README “Docs — Start Here” order → `Overview → Notation → Model → State → Trigger → Beyond → T&E`
- [ ] Docs/Overview.md: move “What To Read First” under TL;DR and emphasize Notation early
- [ ] Add one‑line rationale in README and Overview for the order change

## 3) Stabilize Pointers and Reduce Brittleness (P0)
- [ ] Replace most `file:line` references with “file + function/rule/template name” across docs (Model, Trigger, Notation, Beyond, T&E)
- [ ] Keep a few verified anchors where line numbers add clear value; state this policy at the top of each affected doc
- [ ] Add a “Code Index” or “References” section per doc to park detailed pointers

## 4) Correct Specific References in State.md (P0)
- [ ] Update incorrect line numbers for generic templates:
  - [ ] `candidate` → `CSP-Rules-Generic/GENERAL/templates.clp:79`
  - [ ] `g-candidate` → `.../templates.clp:101`
  - [ ] `is-csp-variable-for-label` / `is-csp-variable-for-glabel` → `:131` / `:139`
  - [ ] `is-typed-csp-variable-for-label` / `is-typed-csp-variable-for-glabel` → `:147` / `:157`
  - [ ] `candidate-in-focus` → `:237`
  - [ ] `context` → `:256`
  - [ ] `chain` / `typed-chain` → `:284` / `:305`
- [ ] Prefer “file + template name” where possible to avoid drift

## 5) Notation Improvements (P1)
- [ ] Move “End‑to‑End Excerpts” near the top; cross‑link from README/Overview
- [ ] Confirm helper file references (existence and names):
  - [ ] `CSP-Rules-Generic/GENERAL/generic-output.clp`
  - [ ] `CSP-Rules-Generic/GENERAL/parameters.clp`
  - [ ] `SudoRules-V20.1/GENERAL/nrc-output.clp`
- [ ] Add a brief “How to read a chain line in 30 seconds” callout

## 6) Editorial Prefaces and Clarity (P1)
- [ ] Add “Before you start / You will learn” prefaces to: Model, State, Trigger, T&E
- [ ] Trim early dense pointer sections; move deep details to the end under “References”
- [ ] Add cross‑links among docs for smooth navigation

## 7) Cross‑Reference Audit (P1)
- [ ] Grep all docs for backticked `file:line` style references; fix or convert
- [ ] Spot‑check app‑specific examples (Map, Kakuro, Latin, Slitherlink, Futoshiki, Hidato, Numbrix) to ensure claimed files exist

## 8) Optional Additions (P2)
- [ ] Add `Docs/Itinerary.md` as a simple roadmap template (if we keep the link)
- [ ] Add a short “Doc maintenance” note explaining the pointer policy

## 9) Execution Order (Tracking)
- [ ] Apply README.md updates
- [ ] Apply Docs/Overview.md updates
- [ ] Apply Docs/State.md corrections
- [ ] Apply Docs/Notation.md edits
- [ ] Sweep pointer conversions across Model/Trigger/Beyond/T&E
- [ ] Final sanity pass: validate links and file references

