# AstroDB Website Directions: conventions for the website workflow

These refine the shared conventions in `astrodb-directions.md` for the **website** skill
(`astrodb-website`). Read `astrodb-directions.md` first — it covers the artifact folder, the
decision-log entry format and what-to-log, the general completion-checklist behavior, and the
"ask, don't assume" rule. This file only states what is specific to the website phase.

## Decision log: `workflow.md`

The website phase reads and appends the pipeline-wide **`workflow.md`** in the user's current working
directory (the project root) — the same log the build phase writes — so the site's setup decisions sit
alongside the build context that produced the database.

- **Read** `workflow.md` at the start if it exists, to carry forward context from earlier phases.
- **Append** one entry (using the standard entry format from `astrodb-directions.md`, newest at the
  **end**) only if the setup involved a notable decision worth recording — a non-default host/port, a
  chosen `--db-path`, or a config the user had to confirm. Purely mechanical setup needs no entry.

## Completion checklist: verify and report, don't persist

The website workflow is a **single** skill, so a persisted `checklists.md` would hold exactly one section
— pure overhead. Follow the shared verify-and-report behavior only: verify every item, reproduce the
evidence-annotated list in your completion message, and never claim an item you didn't verify. Do not
write the checklist out to a separate artifact file.
