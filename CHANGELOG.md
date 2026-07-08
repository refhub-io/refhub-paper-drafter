# Changelog

All notable changes to `refhub-paper-drafter` are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this
project uses [Semantic Versioning](https://semver.org/). History prior to
1.3.0 was not tracked in this file.

## [1.3.0] - 2026-07-08

### Added
- RELATION MAP, a new tracked structure (parallel to SOURCE MAP / FIGURE-TABLE
  INVENTORY) built in Phase 1 from the `relations` field already present in
  the vault export/read response — no new API call. One entry per relation
  (`cites`/`extends`/`builds_on`/`contradicts`/`reviews`/`related`), with a
  `usage` field defaulting to `not yet used`.
- Phase 2e (Related Work Positioning) now cross-references the RELATION MAP
  alongside vault tags — an actual `cites`/`extends`/`contradicts`/etc.
  relation is treated as stronger positioning evidence than tag
  co-occurrence alone.
- Phase 2d (Argument Map) may cite a RELATION MAP entry alongside SOURCE MAP
  entries to justify why a section addresses a gap relative to a specific
  prior item.
- Phase 3's drafting rule notes that a cited RELATION MAP entry can become
  actual prose (e.g. "prior work assumes X; we relax X because Y"), not just
  backstage bookkeeping.
- Phase 5 (R2 Hostile Reviewer) now scans the RELATION MAP for entries still
  marked `usage: not yet used` that plausibly bear on a claim in the draft,
  and flags them the same way an ungrounded claim is flagged.
- When the `refhub` CLI isn't found, the skill now asks the user upfront
  (with the install command included) whether to set it up, instead of
  silently falling back to direct API calls. Declining (or not responding)
  still falls back to direct HTTP calls — this is a nudge, not a hard
  requirement.

### Fixed
- Prerequisites table no longer lists `refhub-skill` as required — it's now
  documented as an optional companion (this skill's own workflow only needs
  `vaults:read`/`vaults:export`), resolving a contradiction with the "works
  standalone without refhub-skill" line elsewhere in the doc.
