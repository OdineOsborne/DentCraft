# DentCraft: plan to make it work, make it clear, then widen it

## Context

Codex worked on DentCraft for a week (11 commits since 4c49469): floating wizard, Expert strip and
multi-tooth settings, library teeth kept at their own size, a contact map, release 0.2.1 and new
branding. The app now has the exocad-style layout, but it is still fragile: the public showcase
screenshot of the Contacts step itself shows "Crown deformation folds or intersects a surface",
and the open failures in AGENTS.md (tooth 35 hang, library fallbacks on 13/16/33, 7 teeth of the
full-arch case, margin detection on 26 and 01/16) are untouched.

The owner wants one plan that (1) makes the program work end to end, then widens it
("horizontal"), starting with **bridges and pontics**; (2) makes the design clear and easy;
(3) uses the reverse-engineering tools, limited by the owner's choice to **behaviour and UI only**
(no code, no proprietary formats); (4) learns from **Atomica CAD**, a second dental CAD that is
installed and running on this machine (`I:\ATOMICA`, Qt6/QML; sample cases in
`I:\Atomica Cases`: a single unit, a 3-unit bridge, a 4-unit bridge), next to exocad (`I:\ExoCAD`).

About Codex's AGENTS.md: keep its rules (owner-authorized screenshots, library dimensions kept).
Its "next work" list is replaced by this plan. Its cautions are treated as advice, not blockers.

## Phase 0 — A scoreboard (1 short task)

Nothing is "working" until one command says so.

- `scratch/tools/scoreboard.sh <exe>`: runs every crown unit of the reference cases through the
  full chain (reuse `scratch/tools/run-case.sh` and `run-real33.sh`), once with the parametric
  library and once with `DENTCRAFT_TOOTH_LIBRARY`, and prints one table: case, teeth designed /
  total, first failing step per tooth, seconds. Saves it as `scratch/runs/scoreboard-<date>.md`.
- Reference set: `scratch/cases/01, 02, 03, 05, 11`, the canine demo, plus the three Atomica
  sample cases converted with `scratch/tools/mkcase.sh` (anonymized, local only).
- Every later task quotes the scoreboard before and after. No task may lower it.

## Phase 1 — Reference study with the reverse tools (behaviour only)

Goal: know exactly how a technician moves through exocad and Atomica, so DentCraft's flow and
defaults are familiar. Output is our own notes, never their code, icons or text.

- **Atomica (running now)**: drive it with computer use on the 1-unit and 3-unit sample cases.
  Record per step: what is on screen, the one thing the user does, the options and their default
  values, how errors are shown, how bridges/pontics/connectors are set up. Export its result STLs
  (local only) to measure: crown size, cement gap, wall thickness, connector area, contact depth.
- **exocad**: same pass from the public wiki (already started, `docs/layout.md`) and, where the
  wiki is thin, by running the installed copy on the same cases.
- **REA tools**: `inspect_desktop_application` / `inspect_pe_imports` for inventory only: module
  list, supported file types, feature names. No decompilation, no reading of QML/XML/library
  formats.
- Deliverables in the repo, in our own words: `docs/reference/workflow.md` (step-by-step
  comparison table: exocad / Atomica / DentCraft / gap), `docs/reference/defaults.md` (parameter
  defaults by material), `docs/reference/bridges.md` (pontic and connector workflow). Measured
  result STLs stay in `scratch/reference/`.

## Phase 2 — Make the single crown always finish

Rule for every step: **automatic first, a manual way out always, never a dead end.**

1. Tooth 35 hang with the library: find the unbounded loop in `build_crown`
   (`crates/dentcraft-design/src/crown.rs`), add a bound and an error.
2. Contacts step folds (`crates/dentcraft/src/contacts_step.rs`, adaptation in
   `dentcraft-design`): when adaptation folds, keep the un-adapted crown, show the distance map and
   a note instead of a red failure; replace the showcase screenshot.
3. Library teeth 13/16/33 fall back: make the fit follow them
   (`anatomy_step.rs::fit`, `dentcraft-anatomy/src/user_library.rs::repair_radial`).
4. Ledge offset inversion on 11/21, 11/24, 03/37: in `offset_margin`, drop inverted offset edges
   and intersect the neighbours (cause table is in `scratch/tools/crown-diag/`).
5. 11/44 (template not radially single-valued), 11/46 (margin not star-shaped: use the area
   centroid as polar origin).
6. Margin detection on 11/26 and 01/16; and make the manual path first-class: when detection
   fails the step switches to "click points along the margin" with the partial result kept.
7. Case 01 side (16 vs 26): ask the owner once.

Target: scoreboard at 100 % of crown units finishing (library tooth or parametric with a note) on
the reference set, each under 60 s.

## Phase 3 — Make the design clear and easy

Driven by the Phase 1 comparison table; done in small slices, each with before/after screenshots.

- **One instruction, one primary action per step** in the wizard panel
  (`crates/dentcraft/src/workflow_panel.rs`): a short sentence, the main button, advanced
  parameters folded under "More". Defaults come from `docs/reference/defaults.md`.
- **Per-tooth status chips** (done / needs check / failed) that jump to the tooth and step when
  clicked; a failed tooth never blocks the others.
- **Messages**: every error says what happened and the next action; recoverable problems are
  notes in the step, not red toasts.
- **Start screen and case setup**: New case → pick teeth and restoration on the chart → load
  scans → Start; recent cases; drag-and-drop scans.
- **Tooth library picker** in the Anatomy step: drop-down with previous/next and a preview, as in
  `docs/layout.md`; library folder chosen in Settings, remembered.
- **Mouse and keys as technicians expect** (right-drag rotate, wheel zoom at pointer, both buttons
  pan, middle click sets the rotation centre), shortcuts shown in tooltips.
- **Owner walkthrough** after each slice: the owner designs one crown from zero; every place he
  hesitates becomes the next fix.

## Phase 4 — Widen: bridges and pontics

Using `docs/reference/bridges.md` and cases 03 (3-unit), Atomica's 3-unit and 4-unit bridges.

1. Pontic anatomy from the tooth library (same loader), placed on the gap between abutments
   (`StepId::PonticBase` exists in `crates/dentcraft-workflow`; `RestorationKind::Pontic` in
   `dentcraft-case`).
2. Pontic base: offset from the gingiva scan with a chosen base type and clearance.
3. Connectors: automatic between neighbours, cross-section handle, **area check against the
   material's minimum** (`Material.min_connector`, already in the case panel).
4. Common insertion axis for the bridge's abutments (extend `dentcraft-design/src/axis.rs`).
5. Merge units into one watertight bridge; export one STL; scoreboard gains "bridges finished".

## Phase 5 — Trust and release 0.3

Quality checks before export (watertight, self-intersection, minimum thickness, connector area,
margin fit), the design report (HTML/JSON), then release 0.3 with the owner's approval.

## How the work is run

- Order: 0 → 1 and 2 in parallel → 3 → 4 → 5. Phases 2–4 are lists of small single-purpose
  tasks, one branch each.
- **Codex does the building** (one job at a time, handoffs that quote file and line); Claude
  plans, writes handoffs, runs the reference study, re-runs the gates and reviews. Claude stays
  under the 70 % weekly cap.
- AGENTS.md "State and next work" is rewritten to point at `docs/plan.md` (a copy of this plan),
  so Codex and Gemini follow the same list.
- Data rules unchanged: cases, libraries, Atomica/exocad outputs stay in `scratch/`; public
  screenshots only with the owner's say.

## Verification

- `cargo fmt --all --check && cargo clippy --workspace --all-targets -- -D warnings && cargo test --workspace`
  green before any merge (shared `CARGO_TARGET_DIR=F:/Ai/DentCraft-wt/target-shared`).
- `scratch/tools/scoreboard.sh` before and after every task; the table goes in the task report.
- UI slices: dark and light screenshots through `--screenshot`, looked at before merge; the
  owner's walkthrough at the end of each Phase 3 slice.
- Bridges: exported bridge is one closed mesh; connector areas ≥ the material minimum; compared
  by measurement with the Atomica/exocad result on the same case.
