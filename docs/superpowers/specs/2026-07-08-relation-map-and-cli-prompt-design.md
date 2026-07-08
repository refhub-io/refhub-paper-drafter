# Relation map and CLI-install prompt — design

Two additions to `skills/refhub-paper-drafter/SKILL.md` (mirrored into `AGENTS.md`): a new tracked structure for using vault relations as drafting evidence, and a CLI-install prompt matching the one added to `refhub-skill`.

## 1. RELATION MAP (new, Phase 1)

**Problem:** Phase 1 already fetches relations as a side effect of `refhub export --vault <id> --format json` (the vault-read response includes `relations` alongside items/tags), but nothing in the skill ever reads or uses that field. Phase 2e (Related Work Positioning) derives intellectual threads from vault *tags* only — it never looks at the actual relationship graph between papers (`cites`, `extends`, `builds_on`, `contradicts`, `reviews`, `related`), even though that graph is exactly what "which prior work does this build on, and what does it leave open" is asking about.

**Fix:** add a new tracked structure, parallel to the existing SOURCE MAP / FIGURE-TABLE INVENTORY / SUPPLEMENTARY MATERIAL ENTRY pattern, rather than overloading SOURCE MAP entries (which are single-claim/single-source and don't naturally represent a relationship *between* two items):

```
RELATION MAP ENTRY:
  id:            RM-001
  from_item:     <vault item id or citation_key>
  to_item:       <vault item id or citation_key>
  relation_type: [cites | extends | builds_on | contradicts | reviews | related]
  usage:         <where this informs the draft, once assigned — gap statement,
                  contradiction discussion, "not yet used" if unassigned>
```

**Build step (Phase 1):** when vault data is fetched (via `refhub export --vault <id> --format json` or the direct `GET /vaults/:vaultId` fallback), read the `relations` field already present in that response and build one RELATION MAP entry per relation. This requires no new API call — it's a new *use* of data Phase 1 already receives. Skip silently (not an error) if the field is absent or empty; relations are enhancement, not a requirement, matching how `notes` is treated as present-when-available.

**Where it's used (matches "full SOURCE MAP integration" — not Related-Work-only):**
- **Phase 2e (Related Work Positioning)** — primary use. When identifying intellectual threads and stating gaps, cross-reference the RELATION MAP for the SOURCE MAP items in scope: a `contradicts` or `extends` relation between two vault items is stronger, more precise positioning evidence than tag co-occurrence alone.
- **Phase 2d (Argument Map)** — a section's supporting SOURCE MAP entries may be strengthened by citing a relevant RELATION MAP entry to justify why the section addresses a gap relative to a specific prior item.
- **Phase 3 drafting** — a `contradicts`/`extends` entry can become actual cited prose ("prior work assumes X; we relax X because Y (SM-###, per RM-###)"), not just backstage bookkeeping.
- **Phase 5 (R2 Hostile Reviewer)** — the existing "Related work: missing influential citations, mischaracterized prior work" attack surface gets a concrete cross-check: scan the RELATION MAP for entries with `usage: not yet used` that plausibly bear on a claim in the draft, and flag them the same way an ungrounded claim gets flagged.

**Print step:** alongside the existing SOURCE MAP summary line in Phase 1, print "RELATION MAP: N entries from vault relations" (or omit the line entirely if N is 0 — don't announce an empty, unused structure).

## 2. CLI-install prompt (Operating RefHub section)

**Problem:** the current "Operating RefHub" section silently falls back to direct API calls when `which refhub` fails — no mention that a CLI exists at all. Separately, the "Prerequisites" table lists both `refhub` CLI and `refhub-skill` as required, but a line later in the same doc says "This skill works standalone without [refhub-skill]" — an existing internal contradiction. This skill's own API usage is entirely `vaults:read`/`vaults:export` (list vaults, tags, search, export) — nothing that requires `refhub-skill`, which exists for the fuller read/write surface (adding items, PDFs, Semantic Scholar).

**Fix:**
- **Prerequisites table:** demote `refhub-skill` from a required row to a one-line "optional companion, for the fuller surface" note below the table — resolving the contradiction rather than perpetuating it.
- **Operating RefHub, when `which refhub` fails:** ask the user upfront, with the install command included, before falling back — mirroring `refhub-skill`'s own fix:

  > The RefHub CLI isn't installed. It's the recommended way to run RefHub read operations — it handles auth, consistent output, and error formatting for you. Want me to set it up now?
  >
  > ```sh
  > npm install -g @refhub/cli
  > export REFHUB_API_KEY=rhk_<publicId>_<secret>   # get one from the RefHub web UI if you don't have one yet
  > ```
  >
  > If you'd rather not, I'll call the API directly for this session instead.

  Declining (or not responding, or an environment that can't run `npm install`) falls back to the existing direct-API instructions unchanged — this is a nudge, not a hard requirement, matching `refhub-skill`'s behavior exactly.

## Scope

Documentation-only change to `skills/refhub-paper-drafter/SKILL.md`, mirrored into `AGENTS.md`, plus a version bump across the three plugin manifests, matching the pattern established in PR #4/#5. No code, no new phases — the RELATION MAP is a new structure inside existing Phase 1, referenced from Phases 2d/2e/3/5; the CLI prompt is a new paragraph inside the existing "Operating RefHub" section.
