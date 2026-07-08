# Relation Map and CLI-Install Prompt Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a RELATION MAP structure to `skills/refhub-paper-drafter/SKILL.md` so vault relations (cites/extends/builds_on/contradicts/reviews/related) become usable drafting evidence throughout the workflow, not just an unused field in the vault export response. Add a CLI-install prompt to the "Operating RefHub" section matching the one already shipped in `refhub-skill`, and fix an existing contradiction in the Prerequisites table. Mirror every change into `AGENTS.md`, bump the version across all three plugin manifests.

**Architecture:** Documentation-only change to one file (`skills/refhub-paper-drafter/SKILL.md`), regenerated into its near-mirror `AGENTS.md` (which differs only in the frontmatter/opening two lines), plus a version bump. No code, no new phases — the RELATION MAP is a new structure inside existing Phase 1, cross-referenced from Phases 2d/2e/3/5; the CLI prompt is a new paragraph inside the existing "Operating RefHub" section.

**Tech Stack:** Markdown only. Verification is manual re-reading/`grep`, not an automated test suite (this repo has none).

## Global Constraints

- Every edit lands in `skills/refhub-paper-drafter/SKILL.md` first, then is mirrored into `AGENTS.md` — do not edit `AGENTS.md` independently or let the two drift.
- The RELATION MAP requires no new API call — it's read from the `relations` field already present in the vault export/read response Phase 1 fetches today.
- Relations are an enhancement, not a requirement: absent/empty `relations` data is a silent no-op, matching how `notes` is already treated as present-when-available.
- The CLI-install prompt is a nudge, not a hard requirement: declining (or an environment that can't run `npm install`) falls back to the existing direct-API instructions unchanged.
- Version bump: `1.2.0` → `1.3.0` across `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`, and both version fields in `.claude-plugin/marketplace.json` (`metadata.version` and `plugins[0].version`).
- This repo has no `CHANGELOG.md` — do not add one; it wasn't requested for this repo. **Superseded during execution:** the user explicitly requested a `CHANGELOG.md` after this plan was written, so Task 5's commit history includes an additional "Add CHANGELOG.md" commit beyond what this plan originally specified.

---

### Task 1: Add the RELATION MAP schema and Phase 1 build step

**Files:**
- Modify: `skills/refhub-paper-drafter/SKILL.md:96-123` (Phase 1, after the SOURCE MAP block)

**Interfaces:**
- Produces: a `RELATION MAP ENTRY` schema block (fields: `id`, `from_item`, `to_item`, `relation_type`, `usage`) that Tasks 2 and 3 will reference by name. The `usage` field's unassigned/default value is literally the string `"not yet used"` — Task 3's Phase 5 step scans for exactly that string, so this exact wording must be preserved.

- [ ] **Step 1: Add the RELATION MAP build step and schema**

In `skills/refhub-paper-drafter/SKILL.md`, find this exact text (it appears once, in Phase 1, right after the "Collect" bullet list and before "**Build the SOURCE MAP**"):

```
There is no required `refhub synthesis` command in this workflow. Any synthesis is agent-produced from local notes plus `items search` or exported vault data, and must cite SOURCE MAP entries just like drafted prose.

**Build the SOURCE MAP** — a structured index with one entry per usable fact or claim:
```

Replace it with:

```
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
                  Discussion" — or not yet used if not yet referenced>
```

If the `relations` field is absent or empty, skip this step silently — relations are an enhancement, not a requirement, the same way `notes` is treated as present-when-available.

**Build the SOURCE MAP** — a structured index with one entry per usable fact or claim:
```

- [ ] **Step 2: Update the Phase 1 summary line to include the RELATION MAP count**

In the same file, find this exact text:

```
Print a summary when complete: "SOURCE MAP: N entries from M local files + K vault queries."
```

Replace it with:

```
Print a summary when complete: "SOURCE MAP: N entries from M local files + K vault queries." If the RELATION MAP has at least one entry, also print "RELATION MAP: N entries from vault relations." — omit this second line entirely if the RELATION MAP is empty; don't announce an empty, unused structure.
```

- [ ] **Step 3: Verify the edit**

```bash
grep -n "RELATION MAP" skills/refhub-paper-drafter/SKILL.md
```

Expected: matches for "Build the RELATION MAP", the `RELATION MAP ENTRY:` schema block, and the summary-line sentence — 4+ matches, no leftover placeholder text.

- [ ] **Step 4: Commit**

```bash
git add skills/refhub-paper-drafter/SKILL.md
git commit -m "Add RELATION MAP schema and Phase 1 build step"
```

---

### Task 2: Reference the RELATION MAP from Phase 2d and 2e

**Files:**
- Modify: `skills/refhub-paper-drafter/SKILL.md:160-170` (Phase 2d, Phase 2e)

**Interfaces:**
- Consumes: `RELATION MAP ENTRY` schema and field names from Task 1 (must already be committed or present in the working tree before this task starts).

- [ ] **Step 1: Update Phase 2e to use the RELATION MAP, not just tags**

Find this exact text:

```
**2e. Related Work Positioning**
From vault tags:
- Which intellectual threads does this work build on?
- For each thread: what does prior work achieve, and what does it leave open?
- State the gap precisely: "Prior work assumes X; we relax X because Y (SM-###)"
```

Replace it with:

```
**2e. Related Work Positioning**
From vault tags and the RELATION MAP:
- Which intellectual threads does this work build on? Use tag co-occurrence as a starting signal, but check the RELATION MAP for `cites`/`extends`/`builds_on`/`contradicts`/`reviews`/`related` entries between SOURCE MAP items in scope — an actual relation is stronger, more precise positioning evidence than tag co-occurrence alone.
- For each thread: what does prior work achieve, and what does it leave open?
- State the gap precisely: "Prior work assumes X; we relax X because Y (SM-###)" — where a RELATION MAP entry supports the gap statement (e.g. a `contradicts` or `extends` relation), cite it too: "...because Y (SM-###, per RM-###)". Set that entry's `usage` field to describe where it was used.
```

- [ ] **Step 2: Add a RELATION MAP cross-reference to Phase 2d**

Find this exact text:

```
**2d. Argument Map**
For each section:
- What is the one key claim this section must establish?
- Which SOURCE MAP entries (by ID) support that claim?
- Flag any section with fewer than 2 supporting entries — surface to user before drafting.
```

Replace it with:

```
**2d. Argument Map**
For each section:
- What is the one key claim this section must establish?
- Which SOURCE MAP entries (by ID) support that claim?
- If a RELATION MAP entry (by ID) strengthens the claim — e.g. justifying why this section addresses a gap relative to a specific prior item — cite it alongside the SOURCE MAP entries and set that RELATION MAP entry's `usage` field.
- Flag any section with fewer than 2 supporting entries — surface to user before drafting.
```

- [ ] **Step 3: Verify the edit**

```bash
grep -n "RELATION MAP" skills/refhub-paper-drafter/SKILL.md
```

Expected: the two new matches from this task appear in addition to Task 1's matches (6+ total).

- [ ] **Step 4: Commit**

```bash
git add skills/refhub-paper-drafter/SKILL.md
git commit -m "Reference the RELATION MAP from Phase 2d and 2e"
```

---

### Task 3: Reference the RELATION MAP from Phase 3 drafting and Phase 5 R2 review

**Files:**
- Modify: `skills/refhub-paper-drafter/SKILL.md:288-296` (Phase 3, "Drafting rule"), `skills/refhub-paper-drafter/SKILL.md:393-401` (Phase 5, "Attack surface")

**Interfaces:**
- Consumes: `RELATION MAP ENTRY` schema, and the exact string `"not yet used"` as the unassigned-`usage` sentinel, from Task 1.

- [ ] **Step 1: Note in Phase 3 that RELATION MAP entries can become cited prose**

Find this exact text:

```
### Drafting rule

For each section in scaffold order:
1. Draft prose using only SOURCE MAP entries + the scaffold argument map
2. Run Loop A (grounding check)
3. Run Loop B (writing quality)
4. Present the polished section; accept feedback before moving to the next
```

Replace it with:

```
### Drafting rule

For each section in scaffold order:
1. Draft prose using only SOURCE MAP entries + the scaffold argument map. Where a RELATION MAP entry was cited in the Phase 2d/2e scaffold for this section, its `contradicts`/`extends`/`builds_on` relationship can become actual prose (e.g. "prior work assumes X; we relax X because Y"), not just backstage bookkeeping — update that RELATION MAP entry's `usage` field to the section it landed in.
2. Run Loop A (grounding check)
3. Run Loop B (writing quality)
4. Present the polished section; accept feedback before moving to the next
```

- [ ] **Step 2: Extend the Phase 5 "Related work" attack surface to cross-check the RELATION MAP**

Find this exact text:

```
- **Related work:** Missing influential citations, mischaracterized prior work
```

Replace it with:

```
- **Related work:** Missing influential citations, mischaracterized prior work. Scan the RELATION MAP for entries still marked `usage: not yet used` — for each, check whether it plausibly bears on a claim already in the draft; if so, flag it the same way an ungrounded claim gets flagged in Loop A, rather than leaving it silently unused.
```

- [ ] **Step 3: Verify the edit**

```bash
grep -n "RELATION MAP\|not yet used" skills/refhub-paper-drafter/SKILL.md
```

Expected: two new matches from this task, plus all prior matches from Tasks 1–2.

- [ ] **Step 4: Commit**

```bash
git add skills/refhub-paper-drafter/SKILL.md
git commit -m "Reference the RELATION MAP from Phase 3 and Phase 5"
```

---

### Task 4: Fix the Prerequisites contradiction and add the CLI-install prompt

**Files:**
- Modify: `skills/refhub-paper-drafter/SKILL.md:10-19` (Prerequisites), `skills/refhub-paper-drafter/SKILL.md:45-65` (Operating RefHub)

**Interfaces:**
- None (self-contained; doesn't depend on Tasks 1–3).

- [ ] **Step 1: Demote `refhub-skill` from required to an optional companion note**

Find this exact text:

```
## Prerequisites

The following must be installed and authenticated in the user's environment:

| Tool | Repository |
|------|-----------|
| `refhub` CLI (`@refhub/cli`) | https://github.com/refhub-io/refhub-cli |
| `refhub-skill` agent skill (Claude Code / Codex / generic harnesses) | https://github.com/refhub-io/refhub-skill |

This skill does not install or configure these tools.
```

Replace it with:

```
## Prerequisites

Required in the user's environment:

| Tool | Repository |
|------|-----------|
| `refhub` CLI (`@refhub/cli`) | https://github.com/refhub-io/refhub-cli |

This skill's own workflow only needs `vaults:read`/`vaults:export` — nothing that requires the separate [`refhub-skill`](https://github.com/refhub-io/refhub-skill) agent skill (that one covers the fuller read/write surface: adding items, PDFs, Semantic Scholar). `refhub-skill` is an optional companion, not a prerequisite for this skill.

This skill does not install or configure these tools by default — see "Operating RefHub" below for how it offers to help with the CLI specifically.
```

- [ ] **Step 2: Add the CLI-install prompt**

Find this exact text:

```
**If the CLI is not available**, call the API directly:

```text
Base URL: https://refhub-api.netlify.app/api/v1
Authorization: Bearer rhk_<publicId>_<secret>

GET /vaults                                        vaults:read
GET /vaults/:vaultId/tags                          vaults:read
GET /vaults/:vaultId/search?tag=&q=&author=&year=  vaults:read
GET /vaults/:vaultId/export?format=json|bibtex     vaults:export
```
```

Replace it with:

```
**If the CLI is not available**, ask the user before falling back to direct API calls — don't silently skip straight to raw HTTP:

> The RefHub CLI isn't installed. It's the recommended way to run RefHub read operations — it handles auth, consistent output, and error formatting for you. Want me to set it up now?
>
> ```sh
> npm install -g @refhub/cli
> export REFHUB_API_KEY=rhk_<publicId>_<secret>   # get one from the RefHub web UI if you don't have one yet
> ```
>
> If you'd rather not, I'll call the API directly for this session instead.

If the user declines (or doesn't respond, or the environment can't run `npm install`), call the API directly:

```text
Base URL: https://refhub-api.netlify.app/api/v1
Authorization: Bearer rhk_<publicId>_<secret>

GET /vaults                                        vaults:read
GET /vaults/:vaultId/tags                          vaults:read
GET /vaults/:vaultId/search?tag=&q=&author=&year=  vaults:read
GET /vaults/:vaultId/export?format=json|bibtex     vaults:export
```
```

- [ ] **Step 3: Verify the edit**

```bash
grep -n "optional companion\|isn't installed\|npm install -g @refhub/cli" skills/refhub-paper-drafter/SKILL.md
```

Expected: 3 matches, one per inserted phrase, both in their respective sections (Prerequisites, Operating RefHub).

- [ ] **Step 4: Commit**

```bash
git add skills/refhub-paper-drafter/SKILL.md
git commit -m "Fix Prerequisites contradiction, add CLI-install prompt"
```

---

### Task 5: Mirror SKILL.md into AGENTS.md, bump version

**Files:**
- Modify: `AGENTS.md` (regenerate from `skills/refhub-paper-drafter/SKILL.md`, preserving `AGENTS.md`'s own opening two lines)
- Modify: `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`, `.claude-plugin/marketplace.json`

**Interfaces:**
- Consumes: the fully-updated `skills/refhub-paper-drafter/SKILL.md` from Tasks 1–4 (this task must run last).

- [ ] **Step 1: Regenerate `AGENTS.md` from `SKILL.md`**

`AGENTS.md` differs from `skills/refhub-paper-drafter/SKILL.md` only in the first few lines (no YAML frontmatter, a plain `# RefHub Paper Drafter — Agent Instructions` title instead of `# RefHub Paper Drafter`, and an "Invoke it by asking the agent..." sentence in place of the one-line description) — everything from `## Prerequisites` onward is byte-for-byte identical between the two files before Tasks 1–4. Since Tasks 1–4 only edited `SKILL.md`, that shared portion has now diverged. Copy everything from `## Prerequisites` onward in `skills/refhub-paper-drafter/SKILL.md` into `AGENTS.md` at the same point, leaving `AGENTS.md`'s existing first 6 lines (`# RefHub Paper Drafter — Agent Instructions`, the description paragraph, the "Invoke it..." line, and surrounding blank lines) untouched.

- [ ] **Step 2: Verify the mirror**

```bash
diff <(tail -n +7 skills/refhub-paper-drafter/SKILL.md) <(tail -n +7 AGENTS.md)
```

Expected: no output (files identical from that point on).

- [ ] **Step 3: Bump the version to 1.3.0 across all three manifests**

```bash
sed -i 's/"version": "1.2.0"/"version": "1.3.0"/g' .claude-plugin/plugin.json .codex-plugin/plugin.json .claude-plugin/marketplace.json
grep -rn '"version"' .claude-plugin/plugin.json .codex-plugin/plugin.json .claude-plugin/marketplace.json
```

Expected: all four occurrences (one in `plugin.json`, one in `.codex-plugin/plugin.json`, two in `marketplace.json`) now read `"version": "1.3.0"`.

- [ ] **Step 4: Commit**

```bash
git add AGENTS.md .claude-plugin/plugin.json .codex-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "Regenerate AGENTS.md mirror, bump version to 1.3.0"
```

---

## Final check

- [ ] Re-read the full `skills/refhub-paper-drafter/SKILL.md` top to bottom once, checking that Phase 1's RELATION MAP section, Phase 2d/2e's cross-references, Phase 3/5's cross-references, the Prerequisites table, and the Operating RefHub prompt all read coherently together (no leftover contradictions, no dangling references to a structure that isn't actually defined).
- [ ] Confirm `git log --oneline` shows 5 commits (or one squashed set, if executed inline rather than via subagents) covering: RELATION MAP schema, 2d/2e references, 3/5 references, Prerequisites + CLI prompt, AGENTS.md mirror + version bump.
