# AstroDB Ingest Directions: Conventions for the ingest workflow

This document collects the conventions specific to the **ingest** skills (`astrodb-ingest-*`), so they
live in one place instead of being repeated in each `SKILL.md`. Each ingest skill's
`references/astrodb-ingest-directions.md` is a symlink to this file.

For the conventions shared with **every** skill (the artifact-folder layout and the "ask, don't
assume" rule), see `astrodb-directions.md`. This file only covers what differs for ingestion: the
ingest **decision log** and the ingest **completion-checklist** convention.

## The ingest decision log: `ingest-workflow.md`

The ingest skills keep their own decision log, **`ingest-workflow.md`**, inside the ingest artifact
directory (`astrodb-ingest-artifacts/ingest-workflow.md`). It is separate from the build workflow
document — the build and ingest phases are run at different times and shouldn't share one file.

Its purpose is to answer "why did the skill make that decision?" — recording dataset-specific context,
choices, and reasoning that would otherwise disappear after the conversation ends. A future reader (or
a later ingest skill) can open it and understand what was done and why.

### Every ingest skill must

1. **Read** `astrodb-ingest-artifacts/ingest-workflow.md` at the start (if it exists) — to carry
   forward context from prior ingest skills.
2. **Create** it with the standard header if it does not exist yet.
3. **Prepend** one entry after completing its main work — **most recent on top**, each entry **dated**.

### Standard header

```markdown
# AstroDB Ingest Workflow Log

Decisions made during ingestion — what was chosen and why. Newest entry first.
```

### Entry format (prepend above older entries)

```markdown
## <skill-name> — <YYYY-MM-DD>

**Input:** <data file path or description of what was processed>

### Decisions
- **<topic>:** <what was decided> — *because <reason>*

### Open questions
- <anything deferred, unresolved, or left for a later skill>
```

Omit "Open questions" if there are none. Do not edit existing entries; add a new one on top.

### What to log

Log any non-obvious choice a future reader might question — a band that needed an SVO ID confirmed, a
regime derived from a filter, a source that had to be ingested first, an ambiguity the user resolved.
Do **not** log mechanical steps (creating directories, installing packages, checks that passed).

## Completion-checklist convention (ingest: verify, don't persist)

Every ingest `SKILL.md` ends with a `## Completion Checklist`. **Verify every item before reporting the
skill done — but do not write the checklist out to a file.** Unlike the build workflow (which persists
`checklists.md`), the ingest skills deliberately keep the checklist *in the run only*:

- The ingest helpers (`ingest_publication`, `ingest_source`, `ingest_photometry`, …) already **fail
  loudly** when a prerequisite isn't met — a missing reference, an unresolved source, a band absent
  from `PhotometryFilters`. If the ingest succeeded, the important preconditions were met by
  construction, so a persisted "I checked this" file adds bloat without adding safety.
- So: run through the checklist, satisfy every item (or record the user's explicit waiver in
  `ingest-workflow.md`), and reproduce the list — with a one-line evidence note per item — **in your
  completion message to the user**, not in a separate artifact file.

Never tick an item you didn't actually verify. Where an item depends on the user, report what happened
("prompted and the user declined"), never a forced action you didn't take.
