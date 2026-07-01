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
| `refhub-cli` | https://github.com/refhub-io/refhub-cli |
| refhub claude plugin | https://github.com/refhub-io/refhub-claude |
| refhub codex plugin | https://github.com/refhub-io/refhub-codex |

must be installed and authenticated before use. this skill does not configure them.

---

## // install

### claude code

```bash
claude plugin install github:refhub-io/refhub-paper-drafter
```

invoke with `/refhub-paper-drafter`

manifest: `.claude-plugin/plugin.json` — instructions: `CLAUDE.md`

---

### codex

**from github** — add to `~/.agents/plugins/marketplace.json`:

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
  "source": { "source": "local", "path": "./plugins/refhub-paper-drafter" },
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

every sentence in the draft must trace to a source_map entry. gaps surface as `[NEEDS SOURCE: <claim>]` — the agent never fills them silently. `\cite{}` keys come from vault entry ids only.

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
