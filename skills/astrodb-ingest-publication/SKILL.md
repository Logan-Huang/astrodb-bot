---
name: astrodb-ingest-publication
description: "Generate and run a Python script that ingests publications (references/citations) into an AstroDB Publications lookup table. Use this skill when the user says: ingest a publication, add a paper/reference/citation, populate the Publications table, 'add this DOI', 'add this bibcode', or provides a DOI/ADS bibcode and wants it in the database. Also use to backfill or complete missing metadata (bibcode, DOI, description) for reference shortnames that ALREADY exist in a Publications table but are blank — e.g. 'look at Publications and fill everything out'. Also use when a data table's discovery references are missing from Publications and need to be added before sources/photometry/spectra can be ingested. Handles a single publication, a batch of references pulled from a table column, or backfilling an existing table. Works standalone or as the prerequisite step before ingest-sources."
compatibility: python, astrodb_utils
---

# Ingest Publications Skill

Generate and run a Python script that adds publications to the `Publications` lookup
table of an AstroDB SQLite database using `astrodb_utils.publications.ingest_publication`.

Every reference in a database (discovery references for sources, references for photometry,
spectra, parallaxes, etc.) must exist as a row in `Publications` first. This skill is how
those rows get there — either one paper at a time, or in a batch from a table that lists
the references a dataset depends on.

Read `references/ingest_publication_api.md` before starting — it has the full function
signatures, the ADS token setup, the `reference` naming convention, and the common
warnings with fixes.

## How publication ingest works

`ingest_publication` takes a **DOI** or **ADS bibcode**, queries the
[NASA ADS](https://ui.adsabs.harvard.edu/) for the paper's metadata, and writes a row with:

| Column | Meaning | Source |
|--------|---------|--------|
| `reference` | short identifier, e.g. `Rojas12` (required, unique) | auto-generated from authors+year, or supplied |
| `bibcode` | ADS bibcode | from ADS, or supplied |
| `doi` | Document Object Identifier | from ADS, or supplied |
| `description` | usually the paper title | from ADS, or supplied |

Auto-populating `bibcode` / `description` from a DOI requires an **ADS token** (see
Prerequisites). Without one, the user must supply `reference` (and ideally `description`)
by hand and the script runs with `ignore_ads=True`.

## Prerequisites

1. **`database.toml`** (astrodb-template-db layout) — required to load the database.
   Check for it in this order:
   1. A path the user explicitly stated in the conversation
   2. `database.toml` in the current working directory (project root)
   3. **If not found, stop and ask the user for it before continuing.** Do not invent a path.
2. **Package**: `astrodb_utils`
3. **ADS token (recommended)** — needed so `ingest_publication` can pull `bibcode`,
   `doi`, and `description` automatically. Check whether it is set up:
   ```python
   from astrodb_utils.publications import check_ads_token
   check_ads_token()
   ```
   If it is **not** set, don't fail silently. Tell the user their options:
   - Set up a token (one-time): instructions in `references/ingest_publication_api.md`.
   - Proceed **without** ADS: the script will use `ignore_ads=True`, which means the
     user must give a `reference` shortname for each paper (and a `description` if they
     want one). DOI/bibcode are stored as given but nothing is fetched.
4. **Internet** — `ingest_publication` reaches out to ADS when a token is present.

## Step 1: Decide the mode, and gather identifiers

**Single publication** — the user names one paper, DOI, or bibcode.
> e.g. "Add the discovery paper for Rojas et al. 2012, DOI 10.1088/0004-637X/748/2/93"

**Batch from a table** — the user points at a data table (CSV/ECSV/FITS) and wants every
reference in some column to exist in `Publications`. This is the usual lead-in to source
ingestion: the `reference` column holds shortnames like `Burg24`, and they must be in
`Publications` before `ingest_source` will accept them.

**Backfill an existing table** — the `Publications` table already holds `reference`
shortnames, but `bibcode` / `doi` / `description` are blank, and the user wants them
"filled out". This is common when a database was bootstrapped from reference strings alone.
See the dedicated "Backfilling existing references" section below — it is the right path
whenever the rows exist but the metadata is empty.

For a batch, load the table and pull the **unique, non-empty** values from the reference
(or DOI/bibcode) column. Show the user the list and confirm before ingesting:

```python
from astropy.table import Table
data = Table.read("path/to/file.ecsv")   # astropy auto-detects .fits/.csv/.ecsv
print(data.colnames)
refs = sorted({str(r).strip() for r in data["reference"] if str(r).strip()})
print(f"{len(refs)} unique references: {refs}")
```

A column of bare shortnames (e.g. `Burg24`) has no DOI/bibcode to look up — ADS can't help,
so those go in with `ignore_ads=True` and the shortname as `reference`. A column of DOIs or
bibcodes *can* be auto-populated when a token is set. Ask the user which their column holds
if it isn't obvious.

## Step 2: Confirm what gets ingested

Always **deduplicate first**. For each candidate, `find_publication` reports whether it is
already present, so existing rows are reported as "already present" rather than re-ingested:

```python
from astrodb_utils.publications import find_publication
found, result = find_publication(db, doi="10.1088/0004-637X/748/2/93")
# found is True if exactly one match exists already
```

Confirm with the user:
- which identifier type each entry uses (DOI, bibcode, or bare reference shortname),
- for `ignore_ads` entries, the `reference` shortname to use (see the naming convention in
  `references/ingest_publication_api.md` — typically first four letters of the first
  author's last name + two-digit year, e.g. `Smit20`),
- the target `database.toml`.

Pick a short `{LABEL}` for the output script: the input filename without extension for a
batch (e.g. `NearbyGalaxies_Jan2021_PUBLIC`), or the lead reference for a single paper
(e.g. `Rojas12`). If a batch filename is long, ask the user for a short label.

## Step 3: Write `tmp/ingest_{LABEL}_publications.py`

Read `scripts/ingest_publication.py` to understand the pattern — DB load, the
`PUBLICATIONS` list of identifier dicts, the `find_publication` dedup check, the
`ingest_publication` call, logging, and the `SAVE_DB` guard. Then write a **tailored**
script using the confirmed identifiers. Do not copy the template verbatim.

The output script must:
- Call `build_db_from_json(settings_file=SETTINGS_FILE)` with the real `database.toml` path.
- Fill `PUBLICATIONS` with the **actual** DOIs / bibcodes / reference shortnames — never
  placeholder strings. For a batch, generate this list from the table column read in Step 1.
- Set `IGNORE_ADS` correctly: `False` when a token is present and entries have DOI/bibcode;
  `True` when there is no token or entries are bare shortnames (and then each entry needs a
  `reference`).
- Call `find_publication` before each `ingest_publication` so existing rows are skipped, not
  duplicated.
- Set `SAVE_DB = False` for the dry run.
- Use the dry-run message: `"Dry run complete — NOT saved. Set SAVE_DB = True to write the database to JSON files."`

## Step 4: Run the dry run

Run `tmp/ingest_{LABEL}_publications.py` with `SAVE_DB = False`. Report:
- how many publications were **added**,
- how many were **already present** (skipped by the dedup check),
- how many **failed**, with the warning for each (see the table in
  `references/ingest_publication_api.md`),
- confirmation that the database was **not** saved.

If the failures are all "ADS token not set", that's the signal to circle back to
Prerequisite 3 — either set up a token or switch the entries to `ignore_ads=True` with
supplied `reference` shortnames.

## Step 5: Confirm and save

After a clean dry run, ask the user:
> Ingestion preview complete: **X** added, **Y** already present, **Z** failed.
> Save these changes to the database? (Re-runs with `SAVE_DB = True`.)

**Never set `SAVE_DB = True` automatically** — only on explicit user confirmation. Saving
writes the JSON files back out via `db.save_database()`.

---

## Backfilling existing references

Use this path when `Publications` already has the `reference` rows but `bibcode` / `doi` /
`description` are blank and the user wants them completed ("look at Publications and fill
everything out"). Read `scripts/backfill_publications.py` for the template.

**The key constraint:** `ingest_publication` works *from* a DOI or bibcode — a bare
shortname like `Bonaca2020` is not something ADS can resolve on its own. So backfilling has
two phases:

1. **Resolve each shortname to a real identifier.** Turn `Surname+Year` into the actual
   paper's DOI and bibcode. Disambiguate using context the database already gives you —
   e.g. the stream/object each reference is attached to in `Associations`, `Sources`, etc.
   (a `Yang2023` attached to "M3" is the M3 tidal-tails paper, not some other Yang 2023).
   Resolve via NASA ADS (author + year + object), or Crossref by title
   (`https://api.crossref.org/works?query.bibliographic=<title>`), and **show the user the
   resolved reference→DOI/bibcode/title table to confirm before writing anything.** This is
   the step that needs care; getting the wrong paper writes wrong metadata.

2. **Fill the rows.** How depends on the database form:
   - **Standalone `.sqlite`** (e.g. a SIMPLE-style file the user hands you): load it with
     `read_db_from_file(db_name=...)` or connect directly, and run an `UPDATE Publications
     SET bibcode=?, doi=?, description=? WHERE reference=?` for each resolved reference. A
     direct update is the reliable choice here because the row already exists — re-ingesting
     would collide on the `reference` primary key. **Write to a copy of the file**, and if
     SQLite throws a "disk I/O error" on a mounted/network path, do the writes on local disk
     and copy the finished file back.
   - **JSON template layout** (`database.toml` + `data/`): with an ADS token you can let
     `ingest_publication(doi=..., reference=...)` pull bibcode/description from ADS. Check
     `get_db_publication` first; if a blank row already exists, prefer updating it over a
     second insert. Then `db.save_database()` after a confirmed dry run.

Backfill is idempotent: skip any reference whose metadata is already populated (report it as
"already filled"), exactly like the `find_publication` dedup guard in the ingest flow. After
writing, verify **zero** rows still have a NULL `bibcode`/`doi`/`description`.
