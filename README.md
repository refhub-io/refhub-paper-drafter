# refhub-paper-drafter

> `// draft • ground • review`

agent skill for drafting hci and visualization research papers from a refhub vault and local notes. every claim traces to a source. nothing is invented.

**part of the [refhub](https://github.com/refhub-io) ecosystem.**

---

## // what it does

| phase | action |
|-------|--------|
| `collect` | reads local note files + queries vault by tag hierarchy → builds a source_map |
| `scaffold` | interactive back-and-forth to lock framing, contributions, and argument structure before writing |
| `draft` | writes prose from source_map only — ungrounded sentences flagged `[NEEDS SOURCE]`, never filled |
| `quality` | strips ai-giveaways, hollow intensifiers, filler transitions, vague claims |
| `review` | adversarial r2 reviewer loop — severity-rated critiques (minor / major / fatal) + revision |
| `output` | markdown or functional latex → session, local file, or overleaf via git bridge |

---

## // prerequisites

| tool | repo |
|------|------|
| `refhub` CLI (`@refhub/cli`) | https://github.com/refhub-io/refhub-cli |
| `refhub-skill` agent skill (Claude Code / Codex / generic harnesses) | https://github.com/refhub-io/refhub-skill |

must be installed and authenticated before use. this skill does not configure them.

minimum RefHub API scopes: `vaults:read`; add `vaults:export` if using `refhub export`.

---

## // install

### claude code

```bash
claude plugin marketplace add https://github.com/refhub-io/refhub-paper-drafter
claude plugin install refhub-paper-drafter@refhub-paper-drafter
```

invoke with `/refhub-paper-drafter`

manifest: `.claude-plugin/plugin.json` — instructions: `CLAUDE.md`

---

### codex

**from github** — add to the Codex/OpenAI agents plugin marketplace configuration:

```json
{
  "name": "refhub-paper-drafter",
  "source": { "source": "github", "repo": "refhub-io/refhub-paper-drafter" },
  "policy": { "installation": "AVAILABLE", "authentication": "ON_INSTALL" },
  "category": "Productivity"
}
```

**locally** — clone then add to marketplace:

```bash
git clone https://github.com/refhub-io/refhub-paper-drafter ~/plugins/refhub-paper-drafter
```

```json
{
  "name": "refhub-paper-drafter",
  "source": { "source": "local", "path": "~/plugins/refhub-paper-drafter" },
  "policy": { "installation": "AVAILABLE", "authentication": "ON_INSTALL" },
  "category": "Productivity"
}
```

**skill only** — no plugin system:

```bash
cp -r skills/refhub-paper-drafter ~/.codex/skills/
# or cross-runtime:
cp -r skills/refhub-paper-drafter ~/.agents/skills/
```

manifest: `.codex-plugin/plugin.json` — instructions: `AGENTS.md`

---

### copilot cli / gemini cli

```bash
cp -r skills/refhub-paper-drafter ~/.agents/skills/
```

---

### any harness (manual)

```markdown
@path/to/refhub-paper-drafter/skills/refhub-paper-drafter/SKILL.md
```

---

## // grounding

```
// correct
[NEEDS SOURCE: interaction cost scales with hierarchy depth]

// wrong
interaction cost generally increases as hierarchies become more complex
```

every sentence in the draft must trace to a source_map entry. gaps surface as `[NEEDS SOURCE: <claim>]` — the agent never fills them silently.

citation keys use RefHub item `bibtex_key` first, then a stable sanitized item ID fallback. the final `.bib` export must use the same keys referenced by `\cite{}`.

phase 1 uses the real RefHub CLI flow:

```bash
refhub vaults list
refhub tags list --vault <vaultId>
refhub items search --vault <vaultId> --tag <tagId>
refhub export --vault <vaultId> --format json
refhub export --vault <vaultId> --format bibtex
```

there is no `refhub synthesis` command requirement. synthesis is produced by the agent from local notes, item search results, and optional vault exports, and every synthesized claim must cite source_map entries.

source_map entries include citation key, locator, evidence role/type/strength, and manuscript-use fields. final output includes an auditable claim-to-source matrix.

## // submission readiness

the skill is a drafting assistant with explicit readiness gates. a manuscript is not marked submission-ready unless all required gates pass:

- zero unresolved `[NEEDS SOURCE]` markers
- zero unresolved fatal r2 findings and explicit disposition of all major findings
- ethics/irb/privacy and consent status, or a not-applicable rationale
- ai-use disclosure
- reproducibility/materials availability statement
- venue/template/checklist status
- paper/evaluation-type reporting fields
- hci/visualization validity checks
- figure/table inventory with source links, takeaway captions, placeholder status, and alt text

---

## // best practices basis

distilled from best-paper-award winners across hci and visualization research:

```
contribution framing   → specific • bounded • falsifiable
related work           → thread-organized • gap-precise • load-bearing citations
design requirements    → labeled t1–tn / d1–dn / rq1–rqn • traceable to data
evaluation             → pre-stated hypotheses • effect sizes • honest negatives
discussion             → interpretation not re-narration • connect back to requirements
limitations            → specific • scope vs validity • each implies an open question
```

---

## // license

`gpl`
