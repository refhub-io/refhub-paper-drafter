# refhub-paper-drafter

A plugin for drafting HCI and visualization research papers. Takes your scattered notes and RefHub vault entries and produces a grounded, structured manuscript modeled on best practices from award-winning research in human-computer interaction and visualization.

**Part of the [RefHub](https://github.com/refhub-io) ecosystem.**

---

## What it does

1. **Collects your inputs** — reads local note files and queries your RefHub vault by topic tags, building a SOURCE MAP of every available claim and reference
2. **Scaffolds interactively** — back-and-forth dialogue to lock in framing, contributions, argument structure, related work positioning, and target venue before writing anything
3. **Drafts with grounding** — every sentence traces to the SOURCE MAP; ungrounded content is flagged `[NEEDS SOURCE]` rather than silently filled
4. **Reviews writing quality** — strips AI-giveaway phrases, hollow intensifiers, filler transitions, and vague claims; requires precise academic prose
5. **Runs R2 hostile review** — adversarial peer review with severity-rated critiques (minor / major / fatal), then revises the draft to address them
6. **Routes output** — Markdown or functional LaTeX; to the session, a local file, or directly to an Overleaf project via Git bridge

---

## Prerequisites

| Tool | Purpose | Repo |
|------|---------|------|
| `refhub-cli` | Query your research vault | https://github.com/refhub-io/refhub-cli |
| RefHub Claude plugin | Claude Code vault integration | https://github.com/refhub-io/refhub-claude |
| RefHub Codex plugin | Codex vault integration | https://github.com/refhub-io/refhub-codex |

These must be installed and authenticated before using this skill. The skill does not configure them.

---

## Installation

### Claude Code

```bash
claude plugin install github:refhub-io/refhub-paper-drafter
```

Then invoke with:

```
/refhub-paper-drafter
```

Plugin manifest: `.claude-plugin/plugin.json`
Instructions file loaded at session start: `CLAUDE.md`

---

### Codex

The repo ships a full Codex plugin manifest (`.codex-plugin/plugin.json`) with the `interface` block Codex needs for its UI (display name, default prompts, category, capabilities).

**Install from GitHub** (when Codex supports GitHub plugin sources):

Add to your `~/.agents/plugins/marketplace.json`:

```json
{
  "name": "refhub-paper-drafter",
  "source": {
    "source": "github",
    "repo": "refhub-io/refhub-paper-drafter"
  },
  "policy": {
    "installation": "AVAILABLE",
    "authentication": "ON_INSTALL"
  },
  "category": "Productivity"
}
```

**Install locally** (clone):

```bash
git clone https://github.com/refhub-io/refhub-paper-drafter ~/plugins/refhub-paper-drafter
```

Then add to `~/.agents/plugins/marketplace.json`:

```json
{
  "name": "refhub-paper-drafter",
  "source": {
    "source": "local",
    "path": "./plugins/refhub-paper-drafter"
  },
  "policy": {
    "installation": "AVAILABLE",
    "authentication": "ON_INSTALL"
  },
  "category": "Productivity"
}
```

**Install skill only** (no plugin system):

```bash
mkdir -p ~/.codex/skills
cp -r skills/refhub-paper-drafter ~/.codex/skills/
# or use the cross-runtime path:
mkdir -p ~/.agents/skills
cp -r skills/refhub-paper-drafter ~/.agents/skills/
```

Plugin manifest: `.codex-plugin/plugin.json`
Instructions file loaded at session start: `AGENTS.md`

---

### Copilot CLI / Gemini CLI

Use the cross-runtime skills path (recognized by both):

```bash
mkdir -p ~/.agents/skills
cp -r skills/refhub-paper-drafter ~/.agents/skills/
```

---

### Manual (any harness)

Clone the repo and import the skill from your project's instructions file:

```markdown
# In CLAUDE.md / AGENTS.md / GEMINI.md
@path/to/refhub-paper-drafter/skills/refhub-paper-drafter/SKILL.md
```

---

## Usage

When you invoke the skill, the agent will:

1. Ask for your local note file paths and RefHub vault tags
2. Build a SOURCE MAP from all collected material
3. Work through the scaffold interactively (framing, contributions, argument map, venue)
4. Ask how you want the output: Markdown or LaTeX, and where (session / file / Overleaf)
5. Draft section by section, running grounding and quality checks
6. Run the R2 adversarial review and revise

The draft is gated on the scaffold approval — no prose is written until you confirm the structure.

---

## Grounding guarantee

The skill enforces a hard grounding constraint: every sentence in the draft must trace to a SOURCE MAP entry. Sentences without a source get flagged `[NEEDS SOURCE: <claim>]` and surfaced to you for resolution. The agent never fills gaps silently.

`\cite{}` keys in LaTeX output come exclusively from vault entry IDs. No keys are invented.

---

## Best practices basis

The writing conventions baked into the scaffold dialogue and quality loop come from structural and rhetorical analysis of best-paper-award winners across HCI and visualization research, covering:

- Contribution framing and falsifiability
- Related work positioning and gap articulation
- Design requirement labeling (T1–Tn, D1–Dn, RQ1–RQn)
- Evaluation design and reporting standards
- Discussion vs. results separation
- Limitations specificity

---

## License

MIT
