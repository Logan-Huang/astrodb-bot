---
name: ingest-sources
description: "Generate and run a Python script that ingests sources (astronomical objects) into an AstroDB Sources table from a data table. Use this skill when the user says: ingest sources, ingest objects, add new sources to the database, add objects to SIMPLE, or provides a FITS/CSV/ECSV file and wants to populate the Sources table. Works standalone or as the step after match-schema."
compatibility: python, astropy, astrodb_utils, astroquery
---

# Ingest Sources Skill

Generate and run a Python script that ingests rows from a data table into the `Sources`
table of an AstroDB SQLite database using `astrodb_utils.sources.ingest_source`.

Read `references/ingest_source_api.md` before starting — it has the full signature,
parameter meanings, and common warnings with fixes.

## Prerequisites

1. **Database**: JSON data files + `database.toml` (astrodb-template-db layout).
   If absent, run the `create-astrodb` skill first.
2. **Packages**: `astrodb_utils`, `astropy`, `astroquery`
3. **Data table**: FITS, CSV, ECSV, or any astropy-readable format, with at minimum
   a source name column and a discovery reference column.
4. **Publications populated**: every reference value must already exist in `Publications`.
   If not, tell the user to run `ingest_publication` first.
5. **Internet (recommended)**: used by `ingest_source` to query SIMBAD when RA/Dec
   are not in the table.

## Required Inputs
1. Path to the data table file (CSV, ECSV, FITS, etc.)
2. Path to `database.toml` — check in order:
   1. A path the user explicitly stated in the conversation
   2. `database.toml` in the current working directory (root of the project)
   3. If not found, ask the user for the path before continuing

---

## Step 1: Load and inspect the data table

```python
from astropy.table import Table
data = Table.read("path/to/file.fits") # astropy auto detects .fits, .csv, .ecsv
# If auto-detect fails: Table.read(..., format="fits")
print(data.colnames)
print(data[:3])
```

Show the user the **column names**, **dtypes**, and a **3-row preview** so they can confirm mapping in the next step.

---

## Step 2: Confirm column mappings
Ask the user to confirm two things: **(A) input file columns** and **(B) DB schema column names**.

### A. Input file columns
 
Present the actual column names from Step 1 — **do not assume defaults**.

| Role | Required? | Notes |
|------|-----------|-------|
| Source name | **Yes** | String column |
| Discovery reference | **Yes** | Must already exist in `Publications` |
| RA (decimal degrees) | No | If absent → SIMBAD fallback |
| Dec (decimal degrees) | No | If absent → SIMBAD fallback |
| Epoch | No | e.g. `"2000.0"` |
| Equinox | No | e.g. `"J2000"` |
| Comment | No | |
| Other reference | No | |

After confirmation, read the first value of the reference column — use it as `{REF}`
to name the output script (e.g. `Burg24`).

Example prompt to user:
> The table has these columns: `Name, RA, Dec, Dist, Reference`
> Which column is the source name? Which is the discovery reference?

### B. DB schema column names
 
These are the column names **in the database `Sources` table** — not the input file.
They vary by database. **Always ask the user which DB they are targeting**, then use the
known defaults for that DB:
 
| Database | ra_col_name | dec_col_name | epoch_col_name |
|----------|-------------|--------------|----------------|
| astrodb-template-db | `ra_deg` | `dec_deg` | `epoch_year` |
| SIMPLE-db | `ra` | `dec` | `epoch` |
| Unknown | **ask the user** | **ask the user** | **ask the user** |
 
To confirm for an unknown DB, check the schema with:
```python
print(db.metadata.tables["Sources"].columns.keys())
```
**Example prompt:**
> Which database are you ingesting into — SIMPLE-db, astrodb-template-db, or another?
> (This determines the column names used internally for RA, Dec, and epoch.)


---

## Step 3: Write `tmp/ingest_{REF}_sources.py`

Fill in all values from Steps 1–2 and write the script to `tmp/ingest_{REF}_sources.py`.
Every variable below must contain a real value — never write placeholder text to the file.

```python
import logging
from astropy.table import Table
from astrodb_utils import build_db_from_json
from astrodb_utils.sources import ingest_source

logging.getLogger("astrodb_utils").setLevel(logging.INFO)
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)
logging.basicConfig(format="%(levelname)s - %(message)s")

SAVE_DB = False  # set True only after a clean dry run

# Adjust to match your project layout
SCHEMA_PATH   = "tests/astrodb-template-db"
DB_NAME       = "tests/astrodb-template-tests"
SETTINGS_FILE = "database.toml"

db = build_db_from_json(
    settings_file=SETTINGS_FILE,
    base_path=SCHEMA_PATH,
    db_name=DB_NAME,
)

# --Load data table--
TABLE_PATH    = "path/to/file.fits"  # confirmed in Step 1
data = Table.read(TABLE_PATH)
logger.info(f"Loaded {len(data)} rows from {TABLE_PATH}")

# --- Column mapping — filled from Step 2 confirmation ---
# Use the ACTUAL column names from your file (not assumed defaults)
SOURCE_COL    = "Name"               # confirmed in Step 2 — required
REFERENCE_COL = "Reference"         # confirmed in Step 2 — required
RA_COL        = "RA"                 # confirmed in Step 2 — set to None → SIMBAD fallback
DEC_COL       = "Dec"               # confirmed in Step 2 — set to None → SIMBAD fallback

# Optional columns — set to None if not present in table
EPOCH_COL     = None                 # optional — set to column name if present
EQUINOX_COL   = None                 # optional — set to column name if present
COMMENT_COL   = None                 # optional — set to column name if present
OTHER_REF_COL = None                 # optional — set to column name if present

# --- DB schema column names — confirmed in Step 2B ---
# These are column names IN the database Sources table, not the input file.
# astrodb-template-db defaults: ra_deg, dec_deg, epoch_year
# SIMPLE-db uses:               ra,     dec,     epoch
# To check your DB: print(db.metadata.tables["Sources"].columns.keys())
RA_COL_NAME    = "ra_deg"
DEC_COL_NAME   = "dec_deg"
EPOCH_COL_NAME = "epoch_year"

# Ingest Loop
sources_added = sources_skipped = 0
for row in data:
    source_name = str(row[SOURCE_COL])
    try:
        ingest_source(
            db,
            source=source_name,
            reference=str(row[REFERENCE_COL]),
            ra=float(row[RA_COL]) if RA_COL else None,
            dec=float(row[DEC_COL]) if DEC_COL else None,
            epoch=str(row[EPOCH_COL]) if EPOCH_COL else None,
            equinox=str(row[EQUINOX_COL]) if EQUINOX_COL else None,
            other_reference=str(row[OTHER_REF_COL]) if OTHER_REF_COL else None,
            comment=str(row[COMMENT_COL]) if COMMENT_COL else None,
            ra_col_name=RA_COL_NAME,
            dec_col_name=DEC_COL_NAME,
            epoch_col_name=EPOCH_COL_NAME,
            raise_error=True,
        )
        sources_added += 1
        logger.info(f"Ingested: {source_name}")
    except Exception as e:
        sources_skipped += 1
        logger.warning(f"Skipping {source_name}: {e}")
 
logger.info(f"Done: {sources_added} ingested, {sources_skipped} skipped out of {len(data)} rows")

if SAVE_DB:
    db.save_database(directory="data/")
    logger.info("Database saved to data/")
else:
    logger.info("Dry run complete — NOT saved. Set SAVE_DB = True to persist.")
```

---

## Step 4: Run the script

Run `tmp/ingest_{REF}_sources.py` with `SAVE_DB = False`. Report:

-  How many sources were ingested successfully
- Any rows skipped with their warning messages
- Confirmation that the database was **not** saved

See `references/ingest_source_api.md` for the common warnings table and how to fix each one.


## Step 5: Confirm and save

After a successful dry run, ask the user:
> Ingestion preview complete: **X** ingested, **Y** skipped out of **Z** rows.
> Would you like to save these changes to the database? (Re-runs with `SAVE_DB = True`)

**Never set `SAVE_DB = True` automatically** — only on explicit user confirmation.

## Key Behaviors
 
1. **Missing RA/Dec**: if `RA_COL = None`, `ingest_source` queries SIMBAD automatically.
   If SIMBAD has no match, that row is skipped with a warning.
2. **Duplicate sources**: if a source already exists, `ingest_source` adds the new name
   as an alternate in `Names` — it does not re-insert into `Sources`.
3. **Missing reference**: `reference` must already be in `Publications` or ingestion fails.
   Remind the user to run `ingest_publication` first.
4. **Unicode dashes**: handled automatically by `ingest_source`
   (en dash, em dash, minus sign, figure dash → `-`).
5. **DB schema column names**: defaults (`ra_deg`/`dec_deg`/`epoch_year`) match
   astrodb-template-db. SIMPLE-db uses `ra`/`dec`/`epoch`. Wrong values cause all rows
   to silently skip — always confirm the target DB in Step 2B.