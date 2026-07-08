---
name: refhub-paper-drafter
description: "Draft an HCI or visualization research paper from scattered notes and a RefHub vault. Produces grounded manuscripts modeled on best-paper-award exemplars, with interactive scaffolding, grounding checks, writing quality review, and an adversarial R2 reviewer loop."
---

# RefHub Paper Drafter

Draft HCI and visualization research manuscripts from personal notes and vault entries. Every claim in the draft traces to a source in the collected inputs. Nothing is invented.

## Prerequisites

The following must be installed and authenticated in the user's environment:

| Tool | Repository |
|------|-----------|
| `refhub` CLI (`@refhub/cli`) | https://github.com/refhub-io/refhub-cli |
| `refhub-skill` agent skill (Claude Code / Codex / generic harnesses) | https://github.com/refhub-io/refhub-skill |

This skill does not install or configure these tools.

---

## Operating RefHub

This skill only needs **read** access — resolving vaults/tags, searching items, and exporting vault content for the SOURCE MAP. It never writes to RefHub.

**Auth:** set `REFHUB_API_KEY` (scope `vaults:read`; add `vaults:export` if using `refhub export`):

```sh
export REFHUB_API_KEY=rhk_<publicId>_<secret>
```

**If the `refhub` CLI is available** (`which refhub` succeeds), use it — it handles auth, formatting, and errors:

```sh
refhub vaults list
refhub tags list --vault <vaultId>
refhub items search --vault <vaultId> --tag <tagId>
refhub export --vault <vaultId> --format json
refhub export --vault <vaultId> --format bibtex
```

Exit codes: `0` success · `1` API error · `2` bad arguments · `3` auth error (missing/invalid key).

**If the CLI is not available**, call the API directly:

```text
Base URL: https://refhub-api.netlify.app/api/v1
Authorization: Bearer rhk_<publicId>_<secret>

GET /vaults                                        vaults:read
GET /vaults/:vaultId/tags                          vaults:read
GET /vaults/:vaultId/search?tag=&q=&author=&year=  vaults:read
GET /vaults/:vaultId/export?format=json|bibtex     vaults:export
```

**Errors:**
- `401` — missing/invalid API key. Stop, report clearly, ask the user for a valid key. Do not retry with the same key.
- `403 missing_scope` — report which scope is missing (`vaults:read` or `vaults:export`). Do not attempt a workaround.
- `404` — the vault/tag/item id is wrong. Verify it. Do not guess a replacement or silently skip it.
- **Never infer a `vault_id` from a vault name.** Always resolve it from `GET /vaults` (or `refhub vaults list`) first.

Item metadata returned by `search`/`export` includes a free-text `notes` field where present — treat it as a legitimate SOURCE MAP entry (`evidence_type: paraphrase` or `quote`, citing the item), not just title/authors/abstract.

For the full read/write RefHub surface beyond what this skill needs — adding items, managing tags, PDFs, Semantic Scholar enrichment — install [`refhub-skill`](https://github.com/refhub-io/refhub-skill) alongside this plugin. This skill works standalone without it.

---

## Core Constraint: Grounding

**The draft may only contain content traceable to the SOURCE MAP built in Phase 1.**

- No invented citations.
- Citation keys use this precedence:
  1. RefHub item `bibtex_key` when present.
  2. A stable sanitized RefHub item ID fallback, prefixed if needed so it is a valid BibTeX key.
  3. The final `.bib` file or BibTeX export must use exactly the same keys referenced by `\cite{}`.
- No paraphrased claims that are not in the SOURCE MAP.
- No gap-filling with plausible-sounding text.
- Ungrounded content → `[NEEDS SOURCE: <verbatim claim>]` — flag to user, never draft around it.

---

## Phase 1 — Input Collection

**Ask the user for:**
1. Paths to local note files (Markdown, plain text)
2. RefHub vault name or ID
3. RefHub topic tag names or IDs to query

**Collect:**
- Read each local file in full
- Resolve vault IDs with `refhub vaults list`
- Resolve tag names and hierarchy with `refhub tags list --vault <vaultId>`
- For each selected tag ID, run `refhub items search --vault <vaultId> --tag <tagId>`
- Optionally export source material with `refhub export --vault <vaultId> --format json` for auditable item metadata, notes, abstracts, and relations
- Optionally export BibTeX with `refhub export --vault <vaultId> --format bibtex` before LaTeX output

There is no required `refhub synthesis` command in this workflow. Any synthesis is agent-produced from local notes plus `items search` or exported vault data, and must cite SOURCE MAP entries just like drafted prose.

**Build the RELATION MAP** — read the `relations` field already present in the vault export/read response fetched above (no new API call). One entry per relation:

```
RELATION MAP ENTRY:
  id:            RM-001
  from_item:     <vault item id or citation_key>
  to_item:       <vault item id or citation_key>
  relation_type: [cites | extends | builds_on | contradicts | reviews | related]
  usage:         <where this informs the draft, once assigned — e.g. "gap
                  statement in Related Work", "contradiction discussion in
                  Discussion" — or "not yet used" if not yet referenced>
```

If the `relations` field is absent or empty, skip this step silently — relations are an enhancement, not a requirement, the same way `notes` is treated as present-when-available.

**Build the SOURCE MAP** — a structured index with one entry per usable fact or claim:

```
SOURCE MAP ENTRY FORMAT:
  id:              SM-001
  claim:           <verbatim or close paraphrase>
  source:          <file:path:line | vault:<vaultId>/tag:<tagId>/item:<itemId>>
  citation_key:    <bibtex_key | sanitized item ID | n/a for uncited local note>
  locator:         <page | section | paragraph | timestamp | line | quote span>
  evidence_role:   [prior-work | user-empirical-data | method-artifact | result | design-rationale | limitation | background]
  evidence_type:   [quote | paraphrase | measurement | observation | artifact | citation-metadata]
  evidence_strength: [strong | moderate | weak | context-only]
  manuscript_use:
    section:       <planned section or "unassigned">
    paragraph_id:  <P-### once drafted, or "pending">
    sentence_id:   <S-### once drafted, or "pending">
    use:           [supports-claim | motivates-gap | defines-method | reports-result | frames-limitation | background]
  notes:           <uncertainties, conflicts, or follow-up needed>
```

Print a summary when complete: "SOURCE MAP: N entries from M local files + K vault queries." If the RELATION MAP has at least one entry, also print "RELATION MAP: N entries from vault relations." — omit this second line entirely if the RELATION MAP is empty; don't announce an empty, unused structure.

Emit the SOURCE MAP as an auditable table or JSON block before drafting. The SOURCE MAP is closed after Phase 1. New entries are added only when the user resolves a `[NEEDS SOURCE]` flag in Phase 3; record such additions in a "Source-map amendments" log with date, source, and reason.

---

## Phase 2 — Scaffold (interactive)

**Do not write any prose. Do not proceed to Phase 3 until the user explicitly approves the scaffold.**

Work through these topics interactively, one per exchange:

**2a. Paper Framing**
- What phenomenon or problem does this paper address?
- Who encounters this problem and what is the cost of not solving it?
- Why is now the right time (new data, new scale, new capability)?

**2b. Contributions**
Draft a numbered contribution list. Each contribution must be:
- Specific and bounded: "We contribute X that does Y, demonstrated by Z"
- Not a feature listing: not "we present a novel system"
- Falsifiable: a reader can check whether it was delivered

**Count guidance:** 3–4 contributions is typical for a full paper. A longer list is a signal to consolidate, not a target to fill toward.

**Component-vs-contribution test:** if a candidate contribution is a design or implementation detail that belongs to another contribution already in the list (an encoding choice, an interaction technique that is part of a system already named), it belongs in that contribution's body-text description as a sub-point — not as its own numbered item.

**Delivered vs. Pending:** a contribution describing work not yet complete (e.g., a study that is designed but not yet run) is allowed — drafting happens before every result exists — but must be tagged `[PENDING]` and paired with an evaluation plan. Ask: "For any contribution that isn't fully delivered yet, how do you plan to evaluate or validate it, and what would make it count as delivered?" Record the answer next to the tagged contribution.

Verify that each contribution is grounded in the SOURCE MAP. Once the Evaluation section is scaffolded (2c–2f), check each contribution against it: fully-delivered (untagged) contributions must have matching evidence there; `[PENDING]`-tagged contributions are expected not to have evidence yet — that is what the tag communicates, not a defect to fix.

**2c. Section Order**
Propose a structure based on the contribution type:
- *Problem-driven:* Intro → Related Work → Problem Characterization → Design → Implementation → Evaluation → Discussion → Conclusion
- *Technique-driven:* Intro → Related Work → Approach → Algorithm/System → Results → Evaluation → Limitations → Conclusion
- *Study-driven:* Intro → Related Work → Study Design → Results → Discussion → Implications → Conclusion

Justify the order against the contributions.

**2d. Argument Map**
For each section:
- What is the one key claim this section must establish?
- Which SOURCE MAP entries (by ID) support that claim?
- Flag any section with fewer than 2 supporting entries — surface to user before drafting.

**2e. Related Work Positioning**
From vault tags:
- Which intellectual threads does this work build on?
- For each thread: what does prior work achieve, and what does it leave open?
- State the gap precisely: "Prior work assumes X; we relax X because Y (SM-###)"

**2f. Design Requirements / Task Labels** (where applicable)
- Derive requirements from SOURCE MAP entries (domain observations, expert input, study findings)
- Label explicitly: T1–Tn, D1–Dn, RQ1–RQn
- Each requirement will be used to justify design decisions in Phase 3

**2g. Target Venue and Format**
Ask the user which venue they are targeting (name or describe it). Then ask:
- Does the venue use a LaTeX template? If so, which — or share the `\documentclass` line.
- Common templates: `acmart` (sigconf, manuscript, etc.), `IEEEtran` (journal, conference), custom.
- If the user doesn't know, default to Markdown output until they confirm.

**2h. Paper Type and Length Budget**

Ask the user which paper type they are targeting, since it sets a hard page/space budget that governs what content the manuscript can afford and what must move to supplementary material in Phase 4.

| Venue / track | Paper type | Content budget | References | Total | Notes |
|---|---|---|---|---|---|
| IEEE VIS (TVCG special issue) | Full paper | 9 pages | up to 2 pages (pointers only, not the material itself) | 11 pages | Non-reference content beyond page 9 risks desk rejection |
| IEEE VIS | Short paper | 4 pages | up to 1 page | 5 pages | |
| IEEE TVCG (regular/direct journal submission, outside the VIS cycle) | Journal paper | ~12 pages, author's disposal | counted within the total, not separate | ~12 pages | This figure and the overlength-fee policy shift; confirm against the current TVCG author guidelines before relying on it |
| EuroVis / CGF | Full paper | 10 pages (incl. figures, acknowledgements) | up to 2 pages | ~12 pages | References excluded from the 10-page count |
| EuroVis / CGF | Short paper | 4 pages | +1 page | 5 pages | |
| EuroVis / CGF (STAR) | Survey / State-of-the-Art Report | ~20 pages is the norm, not a hard cap | not capped | — | Initial sketch stage is 2 pages. Literature-search methodology is a required, load-bearing section — see the Survey/STAR reporting fields below |
| Workshop (venue-specific) | Workshop paper | ~6 pages is typical | varies | — | Formats vary widely by workshop; confirm against that workshop's own CFP rather than assuming VIS/EuroVis defaults |

Record the selection as `page_budget: {content: N, references: M, total: N+M or "uncapped"}` and carry it into Phase 3 drafting and the Phase 4 length triage.

If the target venue isn't in this table, or the user isn't certain of the current limits, ask them to paste the relevant CFP page-limit line rather than guessing — page-limit rules change yearly and vary by venue. Treat an assumed number as unverified risk, the same way an ungrounded claim would be treated in the Core Constraint.

**Best-effort live CFP check.** For venues with a known, stable CFP/guidelines URL pattern (IEEE VIS: `https://ieeevis.org/year/<year>/info/call-participation/...`; EuroVis: `https://eurovis.org.uk/author-guidelines/`), you may verify the static table against the venue's current page instead of relying on the table alone:

1. Ask which submission year/cycle the user is targeting if not already stated — don't assume the current calendar year is the submission year; venue cycles often run ahead (e.g. a 2026-dated CFP can govern work submitted in late 2025 or early 2026).
2. `WebFetch` the relevant page once per venue selection — don't re-fetch on every subsequent scaffold question.
3. If a page limit is found, show the user both figures: the static table value and the fetched value, with a quoted snippet and the source URL, and ask which to record as `page_budget`. If they match, say so briefly and proceed with the static value without forcing a redundant confirmation.
4. Treat any fetch failure, timeout, or "no clear page-limit statement found" as a silent no-op — fall back to the static table and the existing "ask the user to paste the CFP line" prompt. Never block Phase 2h on a failed fetch, and never present a failed or ambiguous fetch as if it confirmed anything.
5. For venues not in the static table, skip the fetch entirely — don't guess a URL for an unknown venue.

State the reliability caveat whenever a fetch is used: CFP pages vary in structure year to year and venue to venue (HTML, JS-rendered, or PDF-only); a successful fetch is a corroboration aid, not a guarantee — the user's confirmation is what's actually recorded.

**2i. Paper and Evaluation Type**
Ask which paper/evaluation types apply. More than one may apply:
- Design study or design probe
- System or technique paper
- Empirical user study
- Interview, qualitative, or mixed-methods study
- Expert review
- Field deployment
- Benchmark, dataset, or model evaluation
- Survey, theory, or position paper

For each applicable type, collect the minimum reporting fields before drafting. Only collect the fields for types the user selected in this step — do not request a type's fields, especially the STAR literature-search protocol, for a paper type where it wasn't selected, even if it seems like generally good methodological rigor:
- Human-subject work: recruitment, inclusion/exclusion, consent, compensation, demographics, apparatus/materials, procedure, randomization/counterbalancing where applicable, accessibility/accommodation, and ethics/IRB status.
- Quantitative evaluation: RQs/hypotheses, sample-size rationale or power analysis where applicable, metrics, test/model choice, assumptions, exclusions/missing-data handling, multiple-comparison policy, effect sizes, confidence intervals, and robustness/sensitivity checks.
- Qualitative evaluation: protocol, participant/context description, coder count, coding procedure, adjudication, reliability/reflexivity stance, theme derivation, quote traceability, and quote privacy/de-identification.
- Expert review: expert domains, number of experts, session format, prompts/tasks, analysis/synthesis method, and conflict-of-interest considerations.
- Design study/visualization work: domain collaborator roles, problem characterization, data/task abstraction, visual encoding and interaction rationale, baseline selection/fairness, scalability/performance, ecological validity, accessibility, and deployment/reflection boundaries.
- Dataset/benchmark work: dataset provenance, inclusion/exclusion, licensing, documentation, quality checks, representativeness/bias, de-identification/privacy, splits/tasks/metrics, and availability constraints.
- Survey / State-of-the-Art report (STAR): literature-search protocol (databases/venues queried, search terms, date range, inclusion/exclusion criteria), corpus size before and after filtering, the taxonomy or classification scheme and how it was derived, a coverage-validation step (how completeness was checked), and an explicit statement of the survey's contribution beyond a bibliography — a taxonomy, a gap analysis, a research agenda.

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

**Scaffold gate:** Present the complete scaffold as a structured summary. Ask: "Does this look right? Say 'proceed' to begin drafting, or tell me what to adjust."

---

## Phase 3 — Draft + Two Review Loops

### Output routing (ask before writing anything)

```
FORMAT:
  (a) Markdown
  (b) Functional LaTeX — provide \documentclass line or template name

DESTINATION:
  (1) In-session (stream to conversation)
  (2) Local file — provide path
  (3) Overleaf — provide Git URL (https://git.overleaf.com/<project-id>)

SUPPLEMENTARY MATERIAL FORM (ask only if DESTINATION is 2 or 3):
  (i)   Separate document — its own file, compiled/submitted independently
        (typical for IEEE VIS/EuroVis supplemental PDFs)
  (ii)  In-PDF appendix — a section appended to the same main document
        (some venues now allow this with cross-linking — confirm the target
        venue actually permits it before choosing this over (i))
  (iii) Outline only — no file; report the SUPPLEMENTARY MATERIAL ENTRY
        outline in-session with the moved content inline
```

For local file / Overleaf destinations, before writing anything:
1. List the target directory (or cloned Overleaf project root) and look for an existing supplementary/appendix file — common names: `supplement.tex`, `supplemental.tex`, `appendix.tex`, `appendices.tex`, `sm.tex`, `si.tex`, or Markdown equivalents.
2. If one exists, treat it as the project's real template: preserve its preamble/structure and append new sections into it. Never overwrite it or scaffold a competing one.
3. If none exists and form (i) or (ii) was chosen, scaffold a minimal one — matching the main document's class for an in-PDF appendix, or a lightweight standalone class for a separate document — and confirm the scaffold with the user before writing further content into it.

For Overleaf (option 3):
1. Clone the project into a temporary directory
2. Write or overwrite the main `.tex` file, and the supplementary file identified/scaffolded above if applicable
3. `git add -A && git commit -m "Initial draft — refhub-paper-drafter" && git push`
4. Report the push result; surface authentication errors explicitly

`\cite{}` keys must follow the citation-key precedence in the Core Constraint. Figures use real assets when provided; placeholders are allowed only when clearly marked in the figure/table inventory and not represented as completed evidence.

---

### Drafting rule

For each section in scaffold order:
1. Draft prose using only SOURCE MAP entries + the scaffold argument map
2. Run Loop A (grounding check)
3. Run Loop B (writing quality)
4. Present the polished section; accept feedback before moving to the next

---

### Loop A — Grounding Check

After drafting each section, audit every sentence:
- Does it trace to a SOURCE MAP entry? → record the SOURCE MAP ID in the claim-to-source matrix
- Does it not? → replace with `[NEEDS SOURCE: <verbatim claim>]`

For each `[NEEDS SOURCE]`, surface it explicitly:
> "I flagged this claim as ungrounded: '<claim>'. Can you point me to a source, or should I remove it?"

- User provides source → add to SOURCE MAP, draft the sentence
- User says remove → drop it
- Never silently write around it. Never paraphrase to avoid the flag.

---

### Loop B — Writing Quality

After the grounding check, rewrite for quality. Eliminate:

| Pattern | Example → Fix |
|---------|---------------|
| AI-giveaway verbs | "delves into" → "examines"; "showcases" → "demonstrates" |
| Hollow intensifiers | "highly effective" → quantify or remove |
| Meta-commentary | "This section presents…" → cut, start with the content |
| Filler transitions | "Furthermore," / "Moreover," / "In addition," → restructure or cut |
| Vague claims | "significantly improves" → "reduces error rate by X% (SM-###)" |
| Passive obscuring agency | "it was found that" → "we found" / "participants reported" |

Require:
- Claims bounded where the SOURCE MAP permits quantification
- Design decisions justified against labeled requirements (T1–Tn, D1–Dn)
- Precise academic voice: no hedging without epistemic reason, no filler

---

## Phase 4 — Length and Supplementary Material Triage

Run this after the full draft is assembled and before the R2 loop. Re-run it after any Phase 5 (R2) revision that adds or restructures content — a fix for one fatal finding can push the manuscript back over budget.

**Goal:** fit the manuscript inside the `page_budget` recorded in Phase 2h without cutting content the venue requires, or content the SOURCE MAP can't support anywhere else.

1. **Measure.** For LaTeX output, build and check the real page count. For Markdown or in-session drafts, estimate from word count (roughly 450–550 words per two-column page as a rough proxy) and say plainly that it is an estimate, not a typeset count.
2. **Classify every section, subsection, figure, and table into one of three tiers:**
   - `essential` — required to establish a contribution or pass a Phase 6 gate (core method, primary result, ethics/IRB statement).
   - `supportable` — strengthens the paper, but the core claims survive without it in the main body (extended examples, secondary analyses, full parameter sweeps, extended related-work discussion, pilot detail, full interview protocol, raw qualitative coding tables).
   - `cuttable` — can be shortened or removed with no loss to the argument (redundant transitions, duplicated figures, verbose setup).
3. **If over budget, act in this order:**
   a. Tighten prose using the Loop B patterns before cutting content.
   b. Move `supportable` material to supplementary material — never delete it. Replace it in the main text with a one-sentence pointer: "Full <protocol/results/derivation> in supplementary material (Section S<N>)."
   c. Only cut `cuttable` material outright.
   d. Never move `essential` material to supplementary material just to hit a page count. If the paper still doesn't fit after (a)–(c), say so explicitly and ask the user whether to trim scope/contributions or target a longer-format venue — do not silently over-cut essential content.
4. **Materialize moved content — don't just log it.** Follow the SUPPLEMENTARY MATERIAL FORM chosen in the Phase 3 output-routing step:
   - **Separate document / in-PDF appendix:** write the actual moved prose (not a summary) into the supplementary file identified or scaffolded in Phase 3, under a heading matching its `S-###` id, in the venue's expected numbering (e.g. "S1", "Appendix A"). Update the pointer sentence left in the main draft to reference that exact label.
   - **Outline only (in-session, no file destination):** render the full moved prose inline in the outline entry, not a one-line description — the user still needs the actual content somewhere, not just a record that something moved.
   - Never invent a supplementary template from scratch when the target project already has one — reuse and extend it (Phase 3 detection step), the same way an ungrounded claim is never filled in with a plausible-sounding guess.

   Track every move in the outline:

```
SUPPLEMENTARY MATERIAL ENTRY:
  id:            S-001
  moved_from:    <main-paper section/paragraph id>
  content:       <one-line description>
  file:          <path to supplementary file, or "in-session only">
  label:         <label used at the destination, e.g. "S1" or "Appendix A">
  source_ids:    <SM-### list>
  referenced_at: <main-paper location of the pointer sentence>
  rationale:     <why it's supportable, not essential>
```

5. **Reconcile references separately from content** at venues that cap them separately (VIS full/short, EuroVis full/short). A paper can be within its content budget and still fail on an oversized reference list — trim citations that aren't load-bearing (see the Related Work best practice) before cutting main content to compensate.
6. **Survey / STAR reports:** the ~20-page CGF convention is a norm, not a hard cap. Don't force-fit a survey into a shorter budget — instead confirm the corpus breadth and taxonomy depth match the space the user intends to use.
7. **Report the outcome:**

```
LENGTH TRIAGE
Venue/type: <e.g. IEEE VIS full paper>
Budget: <content>p content + <refs>p references (<total> total, or "uncapped")
Estimated current length: <content>p content + <refs>p references
Status: [within budget | over — moved N items to supplementary | over — needs scope decision]
Moved to supplementary: <S-### list or "none">
Reference list: <count> entries, <est. pages>
```

---

## Phase 5 — R2 Hostile Reviewer Loop

After the full draft is assembled, adopt the R2 persona and attack the paper.

**R2 Persona:** Skeptical, thorough, looking for reasons to reject. Not hostile for sport — hostile because the field has high standards.

**Attack surface:**
- **Novelty:** Does prior work already do this? Is the claimed gap overstated?
- **Methodology:** Is the evaluation sound? Are baselines appropriate and fairly chosen?
- **Claim support:** Do the conclusions follow from the data presented?
- **Evaluation validity:** Sample size, task selection, participant diversity, ecological validity
- **Over-interpretation:** Does the discussion claim more than the evidence allows?
- **Limitations:** Are significant validity threats omitted or understated?
- **Related work:** Missing influential citations, mischaracterized prior work
- **Length/scope discipline:** Was essential material improperly deferred to supplementary in Phase 4 to hit the page budget? Conversely, is the paper padded or under-length for the contribution it claims?

**Output format:**

```
R2 REVIEW

[fatal] #1 — <one-sentence statement of the critique>
<2–3 sentences: what is wrong, what evidence or prior work triggers this>

[major] #2 — ...

[minor] #3 — ...
```

Severity:
- `[minor]` — addressable with a sentence or paragraph; no structural change needed
- `[major]` — requires additional content, restructuring, or substantive qualification
- `[fatal]` — undermines a core contribution; must be resolved before submission

**Resolution:** For each critique (fatal and major first), propose a specific revision. User approves or redirects. Apply to draft. Keep an R2 disposition log with status `[resolved | deferred | unresolved]`, revision location, and remaining risk.

Do not present a draft as submission-ready while any fatal critique is unresolved. Major critiques may be explicitly deferred only if the final readiness report marks the manuscript "not submission-ready" or "submission-ready with unresolved major risk" and explains the risk.

If resolving a critique adds or restructures content, re-run Phase 4 (Length and Supplementary Material Triage) before moving on.

---

## Phase 6 — Submission Readiness Gate

Before final output, produce a readiness report. If any required gate fails, say so plainly and mark the manuscript **not submission-ready**.

Required gates:
- **Grounding:** zero unresolved `[NEEDS SOURCE]` markers; all manuscript claims trace to SOURCE MAP IDs.
- **Provenance:** emit a claim-to-source matrix with manuscript section, paragraph ID, sentence ID, SOURCE MAP ID, source path or vault locator, citation key, evidence role/type/strength, and notes.
- **R2 findings:** zero unresolved fatal findings; every major finding has a disposition and manuscript location or an explicit unresolved-risk note.
- **Ethics/IRB/privacy:** ethics/IRB status or "not applicable" rationale; consent, privacy, de-identification, compensation, and participant/data-log handling where relevant.
- **AI disclosure:** include a venue-appropriate statement that an AI drafting assistant was used, what it did, and that authors verified sources and claims.
- **Reproducibility/materials:** data, code, stimuli, protocols, analysis scripts/notebooks, preregistration, supplemental files, and licenses are available or their absence is justified.
- **Venue/template:** target venue, template status, page/word limit, anonymization requirement, metadata, references, appendix/supplement rules, and required checklist status are recorded.
- **Length & venue budget:** manuscript fits the `page_budget` recorded in Phase 2h (content and references measured separately where the venue splits them); the Phase 4 length-triage status is `within budget`; every supplementary-material pointer resolves to a materialized entry (a real `file` + `label`, not just a description) in the supplementary outline; no `essential`-tier content was deferred to supplementary material.
- **Contributions:** no contribution-list entry carries an unresolved `[PENDING]` tag for a `ready` status; `ready-with-risks` or `not-ready` may still carry unresolved `[PENDING]` tags if the user explicitly accepts that risk.
- **Method/evaluation type:** all fields from Phase 2i applicable to the paper type are complete or explicitly marked unavailable with consequences.
- **HCI/visualization validity:** address construct, internal, external, ecological, conclusion, design-study, task/data abstraction, baseline, visual encoding, interaction, accessibility, and scalability/performance validity where applicable.
- **Figures/tables:** every figure/table has an inventory entry, source IDs, takeaway caption, placeholder status, and alt text.
- **Declarations:** funding, conflicts of interest, acknowledgments, author contributions where required, and data/materials availability statement.

Readiness output:

```
SUBMISSION READINESS
Status: [ready | ready-with-risks | not-ready]

Gate results:
  Grounding: pass/fail — <notes>
  Provenance: pass/fail — <notes>
  ...

Required before submission:
  - <blocking item>

Claim-to-source matrix:
  <table or JSON>
```

---

## Best Practices Reference

Distilled from best-paper-award winners across HCI and visualization research venues. These shape what the scaffold dialogue asks and what the quality loop rewrites.

### Introduction
- Open with a concrete phenomenon, real artifact, or observable problem — not a generic field overview
- State the problem before the solution; let the reader feel the gap before learning it is filled
- Contributions: numbered list, specific and bounded, each falsifiable
- Scope the evaluation immediately: "N participants", "K domain experts", "M datasets"

### Related Work
- Organize by intellectual thread, not chronology
- Each paragraph: names the thread → summarizes its scope → states what it leaves open
- Gap statement is precise: not "this has not been studied" but "prior work assumes X; we relax X because Y"
- Every citation is load-bearing — it supports a specific claim, not background colour

### Design Requirements
- Label explicitly: T1/T2/T3, D1/D2/D3, RQ1–RQn
- Each requirement traces to a domain observation, user study finding, or expert input in the SOURCE MAP
- Requirements are used in Methods to justify design decisions — they are not decorative

### Methods / Approach
- Every design decision justified against a labeled requirement
- Described at the level of reproducibility: a reader can reimplement from the paper
- Distinguish novel contributions from adapted components; cite the adapted ones
- Figures carry argument: each figure illustrates a design decision, not just a screenshot

### Evaluation
- State what each study is designed to test before describing it
- User studies: N, selection criteria, task design, metrics, analysis method
- Expert reviews: domain, number, session format, synthesis method
- Quantitative: effect sizes and confidence intervals, not just p-values
- Qualitative: representative quotes tied to themes, not anecdote
- Statistical reporting: report hypotheses/RQs, sample-size rationale, exclusions, missing data, model/test choice, assumptions, correction policy, effect sizes, confidence intervals, and robustness checks where applicable
- Qualitative reporting: report protocol, coding process, coder/reflexivity details, adjudication, reliability/adequacy rationale, and quote traceability
- Report negative results honestly and interpret them

### Discussion
- Interpret findings — what do results mean, not what were the results
- Connect back to requirements: which met, which partially, which revealed new complexity
- Name unexpected findings and offer a hypothesis, even if tentative
- Do not repeat Results

### Limitations
- Specific: "N participants from one institution" not "future work could expand the study"
- Scope limitations (by design) vs. validity limitations (what the methodology cannot rule out)
- Each limitation implies a specific open question — name it

### Conclusion
- Restate contributions as delivered facts, not aspirations
- No new information
- End with a forward-looking implication, not a laundry list of future work

---

## [NEEDS SOURCE] Protocol

```
TRIGGER:  Any sentence in Phase 3 draft not traceable to SOURCE MAP
ACTION:   Replace sentence with [NEEDS SOURCE: <verbatim claim>]
SURFACE:  Flag explicitly to user — never suppress or batch silently
RESOLVE:  User provides source → add to SOURCE MAP, draft the sentence
          User drops claim → remove from draft
NEVER:    Paraphrase around the gap; invent a plausible source; fill silently
```
