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
| `refhub-cli` | https://github.com/refhub-io/refhub-cli |
| RefHub harness plugin | Claude Code: https://github.com/refhub-io/refhub-claude — Codex: https://github.com/refhub-io/refhub-codex |

This skill does not install or configure these tools.

---

## Core Constraint: Grounding

**The draft may only contain content traceable to the SOURCE MAP built in Phase 1.**

- No invented citations. `\cite{}` keys come from vault entries only.
- No paraphrased claims that are not in the SOURCE MAP.
- No gap-filling with plausible-sounding text.
- Ungrounded content → `[NEEDS SOURCE: <verbatim claim>]` — flag to user, never draft around it.

---

## Phase 1 — Input Collection

**Ask the user for:**
1. Paths to local note files (Markdown, plain text)
2. RefHub vault topic tags to query (hierarchical — query parent tags to surface sub-tag structure)

**Collect:**
- Read each local file in full
- Run `refhub-cli search --tag <tag>` for each topic tag
- Run `refhub-cli synthesis --tag <tag>` to surface the user's own conceptual summaries and synthesis notes
- Capture the vault's tag hierarchy to understand conceptual organization

**Build the SOURCE MAP** — a structured index with one entry per usable fact or claim:

```
SOURCE MAP ENTRY FORMAT:
  ID:      SM-001
  Claim:   <verbatim or close paraphrase>
  Source:  <file:line | vault:tag/entry-id>
  Type:    [empirical | design-rationale | related-work | personal-note | quote]
```

Print a summary when complete: "SOURCE MAP: N entries from M local files + K vault queries."

The SOURCE MAP is closed after Phase 1. New entries are added only when the user resolves a `[NEEDS SOURCE]` flag in Phase 3.

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

`\cite{}` keys sourced exclusively from vault entry IDs. Figures as `\includegraphics{fig-placeholder}` with caption text from the scaffold.

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
- Does it trace to a SOURCE MAP entry? → note the ID internally
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

**Resolution:** For each critique (fatal and major first), propose a specific revision. User approves or redirects. Apply to draft. Present the final revised manuscript in the selected format and destination.

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
