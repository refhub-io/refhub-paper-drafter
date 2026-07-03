# Scaffold Precision Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix four precision gaps in `refhub-paper-drafter`'s scaffold phase, found from real usage: a literature-search-methodology scope leak, contribution-list bloat/overlap/future-tense claims, missing wide-figure review, and no live CFP verification.

**Architecture:** This is a documentation-only skill. All four fixes are text edits inside `skills/refhub-paper-drafter/SKILL.md`'s existing Phase 2 subsections (2b, 2h, 2i, 2j) plus one new Phase 6 gate bullet — no new phases, no code. After all content edits land, `AGENTS.md` is regenerated as a verified mirror and the plugin version is bumped, matching the process already used for PR #4.

**Tech Stack:** Markdown only. No build, no test runner. "Testing" a task means: the exact new text is present (`grep`), the file still renders as valid Markdown (visual check via `Read`), and — for the AGENTS.md task — `diff` confirms byte-for-byte mirroring of the body.

## Global Constraints

- Spec: `docs/superpowers/specs/2026-07-03-scaffold-precision-fixes-design.md` — every task below implements one numbered section of it.
- Version bump: `1.1.0` → `1.2.0` across `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.codex-plugin/plugin.json` (both `version` fields in `marketplace.json`), matching the PR #4 pattern.
- `AGENTS.md` must remain a verbatim mirror of `SKILL.md` from `## Prerequisites` onward, with its own fixed 5-line header (title, blank, description, blank, "Invoke it" line).
- Branch: `feat/scaffold-precision-fixes` (already created off updated `main`, spec commits already on it).
- Push workaround (this machine's SSH is broken): `git -c credential.helper='!gh auth git-credential' -c url."https://github.com/".insteadOf="git@github.com:" push ...`
- One PR for all four fixes (small, related, same review cycle) — not one PR per task.
- No placeholders, no invented page-limit numbers, no invented URLs beyond what the spec already specifies.

---

### Task 1: Fix literature-search-methodology scope leak (Phase 2i)

**Files:**
- Modify: `skills/refhub-paper-drafter/SKILL.md` (Phase 2i, the sentence introducing the type-specific reporting fields list)

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: nothing consumed by other tasks — self-contained sentence-level fix.

- [ ] **Step 1: Make the edit**

Use the Edit tool on `skills/refhub-paper-drafter/SKILL.md`:

old_string:
```
For each applicable type, collect the minimum reporting fields before drafting:
```

new_string:
```
For each applicable type, collect the minimum reporting fields before drafting. Only collect the fields for types the user selected in this step — do not request a type's fields, especially the STAR literature-search protocol, for a paper type where it wasn't selected, even if it seems like generally good methodological rigor:
```

- [ ] **Step 2: Verify**

Run: `grep -n "even if it seems like generally good methodological rigor" skills/refhub-paper-drafter/SKILL.md`
Expected: one match, inside Phase 2i.

- [ ] **Step 3: Commit**

```bash
git add skills/refhub-paper-drafter/SKILL.md
git commit -m "Scope literature-search-protocol reporting fields to STAR-type papers only"
```

---

### Task 2: Contributions tightening (Phase 2b) + Phase 6 gate

**Files:**
- Modify: `skills/refhub-paper-drafter/SKILL.md` (Phase 2b block; Phase 6 required-gates list)

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: the `[PENDING]` tag convention, referenced by the Phase 6 gate added in this same task (kept together since a reviewer needs both halves to evaluate the feature).

- [ ] **Step 1: Replace the Phase 2b block**

old_string:
```
**2b. Contributions**
Draft a numbered contribution list. Each contribution must be:
- Specific and bounded: "We contribute X that does Y, demonstrated by Z"
- Not a feature listing: not "we present a novel system"
- Falsifiable: a reader can check whether it was delivered

Verify that each contribution is grounded in the SOURCE MAP.
```

new_string:
```
**2b. Contributions**
Draft a numbered contribution list. Each contribution must be:
- Specific and bounded: "We contribute X that does Y, demonstrated by Z"
- Not a feature listing: not "we present a novel system"
- Falsifiable: a reader can check whether it was delivered

**Count guidance:** 3–4 contributions is typical for a full paper. A longer list is a signal to consolidate, not a target to fill toward.

**Component-vs-contribution test:** if a candidate contribution is a design or implementation detail that belongs to another contribution already in the list (an encoding choice, an interaction technique that is part of a system already named), it belongs in that contribution's body-text description as a sub-point — not as its own numbered item.

**Delivered vs. Pending:** a contribution describing work not yet complete (e.g., a study that is designed but not yet run) is allowed — drafting happens before every result exists — but must be tagged `[PENDING]` and paired with an evaluation plan. Ask: "For any contribution that isn't fully delivered yet, how do you plan to evaluate or validate it, and what would make it count as delivered?" Record the answer next to the tagged contribution.

Verify that each contribution is grounded in the SOURCE MAP. Once the Evaluation section is scaffolded (2c–2f), check each contribution against it: fully-delivered (untagged) contributions must have matching evidence there; `[PENDING]`-tagged contributions are expected not to have evidence yet — that is what the tag communicates, not a defect to fix.
```

- [ ] **Step 2: Add the Phase 6 gate bullet**

old_string:
```
- **Method/evaluation type:** all fields from Phase 2i applicable to the paper type are complete or explicitly marked unavailable with consequences.
```

new_string:
```
- **Contributions:** no contribution-list entry carries an unresolved `[PENDING]` tag for a `ready` status; `ready-with-risks` or `not-ready` may still carry unresolved `[PENDING]` tags if the user explicitly accepts that risk.
- **Method/evaluation type:** all fields from Phase 2i applicable to the paper type are complete or explicitly marked unavailable with consequences.
```

- [ ] **Step 3: Verify**

Run: `grep -n "PENDING" skills/refhub-paper-drafter/SKILL.md`
Expected: at least 3 matches (Phase 2b explanation, the scaffold question, the Phase 6 gate).

- [ ] **Step 4: Commit**

```bash
git add skills/refhub-paper-drafter/SKILL.md
git commit -m "Tighten contribution-list scaffolding: count guidance, component test, PENDING tag for undelivered work"
```

---

### Task 3: Wide-figure review field (Phase 2j)

**Files:**
- Modify: `skills/refhub-paper-drafter/SKILL.md` (Phase 2j inventory schema)

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: `column_span` field name, referenced informally by Task 4's cross-reference note in the spec (Phase 4 re-check) — no other task edits Phase 4 in this plan, so no code dependency, just a naming consistency note: keep the field name `column_span` if a future plan touches Phase 4.

- [ ] **Step 1: Make the edit**

old_string:
```
**2j. Figure and Table Inventory**
Create an inventory before drafting:

```
FIGURE/TABLE INVENTORY ENTRY:
  id:            <Fig. 1 | Table 1>
  purpose:       <claim or explanation it carries>
  source_ids:    <SM-### list>
  status:        [available | placeholder | needs-data | needs-design]
  caption_takeaway: <one-sentence takeaway>
  referenced_in: <section>
  alt_text:      <accessibility description or "needed">
```
```

new_string:
```
**2j. Figure and Table Inventory**
Create an inventory before drafting:

```
FIGURE/TABLE INVENTORY ENTRY:
  id:            <Fig. 1 | Table 1>
  purpose:       <claim or explanation it carries>
  source_ids:    <SM-### list>
  status:        [available | placeholder | needs-data | needs-design]
  caption_takeaway: <one-sentence takeaway>
  referenced_in: <section>
  alt_text:      <accessibility description or "needed">
  column_span:   [single | double | undecided]
```

For LaTeX two-column templates, mark `column_span: double` (`\begin{figure*}` instead of `\begin{figure}`) when the content warrants it: multi-panel comparisons, wide timelines/sequences, network/map diagrams, matrices/heatmaps with many columns, or full-UI screenshots. Note that `figure*` floats to the top or bottom of a page in a two-column layout — it does not render inline where placed in the source. A late change to `column_span` affects the Phase 4 length estimate; re-check the length triage (Phase 4) when it changes after drafting has started.
```

- [ ] **Step 2: Verify**

Run: `grep -n "column_span" skills/refhub-paper-drafter/SKILL.md`
Expected: at least 3 matches (schema field, the mark instruction, the Phase 4 cross-reference note).

- [ ] **Step 3: Commit**

```bash
git add skills/refhub-paper-drafter/SKILL.md
git commit -m "Add column_span field to figure/table inventory for figure* review"
```

---

### Task 4: Best-effort live CFP check (Phase 2h)

**Files:**
- Modify: `skills/refhub-paper-drafter/SKILL.md` (end of Phase 2h)

**Interfaces:**
- Consumes: nothing from other tasks. Uses the `WebFetch` tool at agent-runtime (not a plan-time dependency).
- Produces: nothing consumed by other tasks.

- [ ] **Step 1: Make the edit**

old_string:
```
If the target venue isn't in this table, or the user isn't certain of the current limits, ask them to paste the relevant CFP page-limit line rather than guessing — page-limit rules change yearly and vary by venue. Treat an assumed number as unverified risk, the same way an ungrounded claim would be treated in the Core Constraint.
```

new_string:
```
If the target venue isn't in this table, or the user isn't certain of the current limits, ask them to paste the relevant CFP page-limit line rather than guessing — page-limit rules change yearly and vary by venue. Treat an assumed number as unverified risk, the same way an ungrounded claim would be treated in the Core Constraint.

**Best-effort live CFP check.** For venues with a known, stable CFP/guidelines URL pattern (IEEE VIS: `https://ieeevis.org/year/<year>/info/call-participation/...`; EuroVis: `https://eurovis.org.uk/author-guidelines/`), you may verify the static table against the venue's current page instead of relying on the table alone:

1. Ask which submission year/cycle the user is targeting if not already stated — don't assume the current calendar year is the submission year; venue cycles often run ahead (e.g. a 2026-dated CFP can govern work submitted in late 2025 or early 2026).
2. `WebFetch` the relevant page once per venue selection — don't re-fetch on every subsequent scaffold question.
3. If a page limit is found, show the user both figures: the static table value and the fetched value, with a quoted snippet and the source URL, and ask which to record as `page_budget`. If they match, say so briefly and proceed with the static value without forcing a redundant confirmation.
4. Treat any fetch failure, timeout, or "no clear page-limit statement found" as a silent no-op — fall back to the static table and the existing "ask the user to paste the CFP line" prompt. Never block Phase 2h on a failed fetch, and never present a failed or ambiguous fetch as if it confirmed anything.
5. For venues not in the static table, skip the fetch entirely — don't guess a URL for an unknown venue.

State the reliability caveat whenever a fetch is used: CFP pages vary in structure year to year and venue to venue (HTML, JS-rendered, or PDF-only); a successful fetch is a corroboration aid, not a guarantee — the user's confirmation is what's actually recorded.
```

- [ ] **Step 2: Verify**

Run: `grep -n "Best-effort live CFP check" skills/refhub-paper-drafter/SKILL.md`
Expected: one match, at the end of Phase 2h.

- [ ] **Step 3: Commit**

```bash
git add skills/refhub-paper-drafter/SKILL.md
git commit -m "Add best-effort live CFP check to Phase 2h with hard fallback to static table"
```

---

### Task 5: Regenerate AGENTS.md mirror and bump version

**Files:**
- Modify: `AGENTS.md` (regenerated body)
- Modify: `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.codex-plugin/plugin.json`

**Interfaces:**
- Consumes: the final state of `skills/refhub-paper-drafter/SKILL.md` from Tasks 1–4.
- Produces: nothing consumed by other tasks — this is the final sync/release-hygiene task.

- [ ] **Step 1: Regenerate AGENTS.md**

```bash
{
  head -n 5 AGENTS.md
  echo
  tail -n +10 skills/refhub-paper-drafter/SKILL.md
} > /tmp/AGENTS.md.new && mv /tmp/AGENTS.md.new AGENTS.md
```

- [ ] **Step 2: Verify the mirror is byte-identical from the body onward**

Run: `diff <(tail -n +10 skills/refhub-paper-drafter/SKILL.md) <(tail -n +7 AGENTS.md)`
Expected: no output (files identical).

- [ ] **Step 3: Bump version in all three manifests**

In `.claude-plugin/plugin.json`, change `"version": "1.1.0"` to `"version": "1.2.0"`.
In `.claude-plugin/marketplace.json`, change both `"version": "1.1.0"` occurrences (`metadata.version` and `plugins[0].version`) to `"1.2.0"`.
In `.codex-plugin/plugin.json`, change `"version": "1.1.0"` to `"version": "1.2.0"`.

- [ ] **Step 4: Verify no stale version string remains**

Run: `grep -rn "1\.1\.0" . --include="*.json" --include="*.md"`
Expected: no output.

Run: `grep -rln "1\.2\.0" . --include="*.json"`
Expected: `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.codex-plugin/plugin.json`.

- [ ] **Step 5: Commit**

```bash
git add AGENTS.md .claude-plugin/plugin.json .claude-plugin/marketplace.json .codex-plugin/plugin.json
git commit -m "Regenerate AGENTS.md mirror, bump version to 1.2.0"
```

---

### Task 6: Push and open PR

**Files:** none (git/GitHub operations only)

**Interfaces:**
- Consumes: all commits from Tasks 1–5 on `feat/scaffold-precision-fixes`.
- Produces: an open PR for user review.

- [ ] **Step 1: Push the branch**

```bash
git -c credential.helper='!gh auth git-credential' -c url."https://github.com/".insteadOf="git@github.com:" push -u origin feat/scaffold-precision-fixes
```

- [ ] **Step 2: Open the PR**

```bash
GH_TOKEN=$(gh auth token) gh pr create \
  --title "Scaffold precision fixes: contribution tightening, literature-search scoping, figure width, live CFP check" \
  --body "$(cat <<'EOF'
## Summary
- Scope the STAR literature-search-protocol reporting fields to Survey/STAR papers only (Phase 2i) — was leaking into full-paper drafts.
- Tighten contribution-list scaffolding (Phase 2b): 3–4 count guidance, a component-vs-contribution test to stop system sub-decisions from being listed as separate contributions, and a `[PENDING]` tag + evaluation-plan question for contributions not yet delivered. Adds a matching Phase 6 gate.
- Add a `column_span` field to the figure/table inventory (Phase 2j) to flag figures that should use `\begin{figure*}` in two-column templates.
- Add a best-effort live CFP check (Phase 2h): for known venues, WebFetch the current CFP page and show it alongside the static table value for user confirmation; hard-falls-back to the existing static-table + user-paste behavior on any fetch failure.
- Regenerate `AGENTS.md` mirror; bump plugin version 1.1.0 → 1.2.0 across all three manifests.

Design spec: `docs/superpowers/specs/2026-07-03-scaffold-precision-fixes-design.md`

## Test plan
- [ ] Re-read Phase 2b, 2h, 2i, 2j end-to-end for internal consistency
- [ ] Run a scaffold with a real full-paper contribution list that has an overlapping/component-style contribution and confirm the agent flags it
- [ ] Run a scaffold with one contribution describing an unfinished study and confirm it gets tagged `[PENDING]` with a recorded evaluation plan, and that Phase 6 blocks `ready` status while it's unresolved
- [ ] Run a scaffold for a non-STAR full paper and confirm the agent does NOT ask for literature-search-protocol details
- [ ] Run a scaffold with a wide/multi-panel figure and confirm `column_span: double` gets suggested
- [ ] Run a scaffold naming IEEE VIS or EuroVis and confirm the agent attempts (or gracefully skips) the live CFP fetch, and never presents a failed fetch as confirmation
- [ ] After merging, run `claude plugin marketplace update refhub-paper-drafter && claude plugin update refhub-paper-drafter` locally and confirm it reports 1.2.0
EOF
)"
```

- [ ] **Step 3: Report the PR URL to the user**

---

## Self-Review Notes

- **Spec coverage:** Task 1 → spec §1. Task 2 → spec §2 (a–e, including the Phase 6 gate). Task 3 → spec §3. Task 4 → spec §4 (a–f). Task 5 → spec "Scope" paragraph (mirror + version bump). All four spec items have a task.
- **Placeholder scan:** no TBD/TODO; every step has literal text to write or an exact command to run.
- **Type consistency:** `column_span` name used consistently between Task 3's schema addition and its own cross-reference sentence (no other task references it, so no drift risk). `[PENDING]` tag used consistently between Task 2's Phase 2b text and its Phase 6 gate bullet.
