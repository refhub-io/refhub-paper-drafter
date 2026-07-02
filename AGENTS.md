# RefHub Paper Drafter — Agent Instructions

Agent-facing instructions for drafting HCI and visualization research manuscripts from personal notes and RefHub vault entries. Every claim in the draft traces to a source in the collected inputs. Nothing is invented.

**Invoke it** by asking the agent to draft a paper, or by typing the skill name in your harness.

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

Print a summary when complete: "SOURCE MAP: N entries from M local files + K vault queries."

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

Verify that each contribution is grounded in the SOURCE MAP.

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

**2h. Paper and Evaluation Type**
Ask which paper/evaluation types apply. More than one may apply:
- Design study or design probe
- System or technique paper
- Empirical user study
- Interview, qualitative, or mixed-methods study
- Expert review
- Field deployment
- Benchmark, dataset, or model evaluation
- Survey, theory, or position paper

For each applicable type, collect the minimum reporting fields before drafting:
- Human-subject work: recruitment, inclusion/exclusion, consent, compensation, demographics, apparatus/materials, procedure, randomization/counterbalancing where applicable, accessibility/accommodation, and ethics/IRB status.
- Quantitative evaluation: RQs/hypotheses, sample-size rationale or power analysis where applicable, metrics, test/model choice, assumptions, exclusions/missing-data handling, multiple-comparison policy, effect sizes, confidence intervals, and robustness/sensitivity checks.
- Qualitative evaluation: protocol, participant/context description, coder count, coding procedure, adjudication, reliability/reflexivity stance, theme derivation, quote traceability, and quote privacy/de-identification.
- Expert review: expert domains, number of experts, session format, prompts/tasks, analysis/synthesis method, and conflict-of-interest considerations.
- Design study/visualization work: domain collaborator roles, problem characterization, data/task abstraction, visual encoding and interaction rationale, baseline selection/fairness, scalability/performance, ecological validity, accessibility, and deployment/reflection boundaries.
- Dataset/benchmark work: dataset provenance, inclusion/exclusion, licensing, documentation, quality checks, representativeness/bias, de-identification/privacy, splits/tasks/metrics, and availability constraints.

**2i. Figure and Table Inventory**
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
```

For Overleaf (option 3):
1. Clone the project into a temporary directory
2. Write or overwrite the main `.tex` file
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

## Phase 4 — R2 Hostile Reviewer Loop

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

---

## Phase 5 — Submission Readiness Gate

Before final output, produce a readiness report. If any required gate fails, say so plainly and mark the manuscript **not submission-ready**.

Required gates:
- **Grounding:** zero unresolved `[NEEDS SOURCE]` markers; all manuscript claims trace to SOURCE MAP IDs.
- **Provenance:** emit a claim-to-source matrix with manuscript section, paragraph ID, sentence ID, SOURCE MAP ID, source path or vault locator, citation key, evidence role/type/strength, and notes.
- **R2 findings:** zero unresolved fatal findings; every major finding has a disposition and manuscript location or an explicit unresolved-risk note.
- **Ethics/IRB/privacy:** ethics/IRB status or "not applicable" rationale; consent, privacy, de-identification, compensation, and participant/data-log handling where relevant.
- **AI disclosure:** include a venue-appropriate statement that an AI drafting assistant was used, what it did, and that authors verified sources and claims.
- **Reproducibility/materials:** data, code, stimuli, protocols, analysis scripts/notebooks, preregistration, supplemental files, and licenses are available or their absence is justified.
- **Venue/template:** target venue, template status, page/word limit, anonymization requirement, metadata, references, appendix/supplement rules, and required checklist status are recorded.
- **Method/evaluation type:** all fields from Phase 2h applicable to the paper type are complete or explicitly marked unavailable with consequences.
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
