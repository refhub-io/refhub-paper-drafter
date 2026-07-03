# Scaffold precision fixes — design

Three corrections to `skills/refhub-paper-drafter/SKILL.md`, found from real usage of the skill (post PR #4) rather than from re-reading the spec cold.

## 1. Literature-search-methodology scope leak (Phase 2i)

**Problem observed:** while drafting a regular IEEE VIS/TVCG/EuroVis full paper, the agent asked for literature-search-protocol details (databases queried, search terms, inclusion/exclusion criteria) — a requirement that Phase 2i's reporting-fields table scopes to the Survey/STAR paper type only. The type-specific field list is introduced with "For each applicable type, collect the minimum reporting fields," which is scoping by omission (don't collect a type's fields if that type wasn't selected) rather than by explicit prohibition. That's evidently not strong enough to stop the model from pattern-matching "rigorous methodology reporting" as generally good practice and pulling it in anyway.

**Fix:** add an explicit negative-scope sentence immediately before the type-specific field list in Phase 2i:

> Only collect the fields for types the user selected in this step. Do not request a type's fields — especially the STAR literature-search protocol — for a paper type where it wasn't selected, even if it seems like generally good methodological rigor.

This mirrors the Core Constraint's existing pattern of treating unrequested rigor the same as an ungrounded claim — appropriate somewhere, not everywhere by default.

## 2. Contributions tightening (Phase 2b)

**Problem observed (concrete example provided by user):** a real draft had 6 numbered contributions for one paper, where contributions #2–#4 were actually design/implementation details of the system already named in #1 (an edge-width delay encoding, a small-multiples timeline, an interaction technique — all components of "MetroScope"), and #6 stated a future result ("results and their implications will be reported once the study is complete") as if already delivered.

**Fix — three additions to Phase 2b:**

a. **Count guidance.** State that 3–4 contributions is typical for a full paper; a longer list is a signal to consolidate, not a target to fill toward.

b. **Component-vs-contribution test.** If a candidate contribution is a design/implementation detail that belongs to another contribution already in the list (an encoding choice, an interaction technique that's part of a named system), it belongs in that contribution's body-text description as a sub-point, not as its own numbered item.

c. **Delivered vs. Pending.** A contribution describing work not yet complete (e.g., a study that is designed but not yet run) is allowed — drafting happens before every result exists — but must be explicitly tagged `[PENDING]` and paired with an evaluation plan. Add a scaffold question: "For any contribution that isn't fully delivered yet, how do you plan to evaluate/validate it, and what would make it count as delivered?" Record the answer next to the tagged contribution.

d. **Verification step (revised).** Once the Evaluation section is scaffolded, check each contribution against it: fully-delivered (untagged) contributions must have matching evidence there; `[PENDING]`-tagged ones are expected not to have evidence yet — the tag is what communicates that, not a defect to fix.

e. **Phase 6 gate addition.** The Submission Readiness Gate checks for any remaining `[PENDING]` contribution tags. A `ready` status is blocked until they're resolved (delivered-and-untagged, or explicitly cut). `ready-with-risks` / `not-ready` may still carry unresolved `[PENDING]` tags if the user explicitly accepts that risk.

## 3. Figure width review (Phase 2j)

**Problem observed:** no mechanism exists for flagging figures that should span both columns (`\begin{figure*}` in a two-column LaTeX template) — wide diagrams, multi-panel comparisons, timelines, matrices, or full-UI screenshots get squeezed into a single-column width by default with no review step to catch it.

**Fix:** add a `column_span: [single | double | undecided]` field to the `FIGURE/TABLE INVENTORY ENTRY` schema in Phase 2j, with heuristics for when `double` is likely warranted: multi-panel comparisons, wide timelines/sequences, network/map diagrams, matrices/heatmaps with many columns, full-UI screenshots. Note in the schema that `\begin{figure*}` floats to the top/bottom of a page in a two-column layout (not inline where placed in source) — this affects the Phase 4 length estimate, so a late change to `column_span` should trigger a re-check of the length triage (cross-reference from Phase 4, not a new phase).

## 4. Best-effort live CFP check (Phase 2h)

**Motivation:** the Phase 2h page-budget table is a static snapshot and says so ("page-limit rules change yearly... treat an assumed number as unverified risk"). For venues already in the table, the agent can try to verify the current year's limit against the venue's live CFP/author-guidelines page instead of relying purely on the static table or a user-pasted line.

**Fix — extend Phase 2h:**

a. **Scope:** only attempt this for venues with a known, stable CFP/guidelines URL pattern already implied by the static table (IEEE VIS `ieeevis.org/year/<year>/info/call-participation/...`, EuroVis `eurovis.org.uk/author-guidelines/`). For venues not in the table, skip the fetch entirely and keep the existing "ask the user to paste the CFP line" fallback — don't guess a URL for an unknown venue.

b. **Year:** ask the user which submission year/cycle they're targeting if not already stated; don't assume the current calendar year is the submission year (cycles run ahead — e.g., a 2026-dated CFP often governs work submitted in late 2025 or early 2026).

c. **Fetch and compare, never silently override.** `WebFetch` the relevant page. If a page limit is found and it differs from — or confirms — the static table value, show the user both: the static table value and the fetched value with a quoted snippet and the source URL, and ask which to record as `page_budget`. If they match, say so briefly and proceed with the static value (no need to force a redundant confirmation click).

d. **Failure handling:** treat any fetch failure, timeout, ambiguous page structure, or "can't find a clear page-limit statement" as a silent no-op — fall back to the static table + existing user-paste prompt. Never block Phase 2h on a failed fetch, and never present a failed/ambiguous fetch as if it confirmed anything.

e. **Reliability caveat, stated to the user whenever a fetch is used:** CFP pages vary in structure year to year and venue to venue (HTML, JS-rendered, or PDF-only); a successful fetch is a corroboration aid, not a guarantee — the user's own confirmation is still what's recorded.

f. **Frequency:** attempt the fetch once per venue selection in a session; don't re-fetch on every subsequent scaffold question.

## Scope

Documentation-only change to one file (`SKILL.md`) plus the mirrored `AGENTS.md` regeneration and a version bump (`1.1.0` → `1.2.0`) across the three plugin manifests, matching the pattern established in PR #4. No code, no new phases — all four fixes land inside existing phases (2b, 2h, 2i, 2j, with cross-references from 4 and 6).
