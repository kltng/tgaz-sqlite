# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SQLite conversion of the TGAZ (Temporal Gazetteer) — the China Historical GIS placename database from Harvard & Fudan University. Contains 82,117 historical placenames with 245,042 spellings spanning 763 BCE to 1911 CE.

**Data sources**: China Historical GIS (77K), Tibetan Buddhist Resource Center (3.6K), Historical Gazetteer of Russia (761).

**Upstream**: MySQL dump from [fccs-dci/containerized_tgaz](https://github.com/fccs-dci/containerized_tgaz). Live API at `https://chgis.hudci.org/tgaz/`.

## Commands

```bash
# Rebuild the database from MySQL dump (requires cloning upstream repo first)
git clone https://github.com/fccs-dci/containerized_tgaz.git /tmp/tgaz_repo
python3 mysql2sqlite.py              # reads from /tmp/tgaz_repo/mysql-init/02-tgaz-dev-2018.sql
python3 mysql2sqlite.py tgaz.db /path/to/dump.sql  # custom paths

# Query the database
python3 query_tgaz.py "北京"
python3 query_tgaz.py "Chang'an" --year -200
python3 query_tgaz.py --fts "dgon"
python3 query_tgaz.py --bbox 108,34,109,35 --year -200
python3 query_tgaz.py --history hvd_70621 --format json
python3 query_tgaz.py --stats

# Alternative: harvest from live API (slower, incomplete — API caps at 200 results/query)
python3 harvest_tgaz.py
```

## Architecture

All scripts are standalone Python 3 with no external dependencies (stdlib only: `sqlite3`, `json`, `re`, `urllib`).

### Database Schema (tgaz.db)

The database preserves the full upstream MySQL relational schema. Key tables:

- **`placename`** — Core table. 82K records with `sys_id` (e.g. `hvd_70621`), coordinates (`x_coord`/`y_coord`), temporal range (`beg_yr`/`end_yr`), feature type FK, data source FK.
- **`spelling`** — 245K name forms (simplified Chinese, traditional Chinese, pinyin, Tibetan, etc.) linked to placenames via `placename_id`. Joined through `script` and `trsys` tables.
- **`part_of`** — 83K jurisdictional parent-child relationships (`child_id` → `parent_id`) with temporal bounds.
- **`mv_pn_srch`** — Pre-joined materialized search view from upstream. Denormalized with name, transcription, coordinates, feature type, parent info. **This is the primary table for most queries.**
- **`ftype`** — Feature type lookup (1,147 types). Fields: `name_vn` (native), `name_tr` (transcribed), `name_en` (English).
- **`data_src`** — Data source lookup. Join key is short code (e.g. `CHGIS`).
- **`snote`** — Historical notes (25K). Linked from `placename.snote_id`.
- **`present_loc`** — Present-day location mappings.
- **`prec_by`** — Predecessor relationships between placenames.

### Views

- **`v_search`** — Denormalized `mv_pn_srch` + `data_src` names. Best for LLM context serialization.
- **`v_placename`** — `placename` + `data_src` + `ftype` joined.
- **`v_placename_names`** — `placename` + `spelling` + `script` + `trsys`.
- **`v_full`** — `placename` + `data_src` + `ftype` + `snote` + `present_loc`.

### FTS5 Indexes

- **`search_fts`** — Names, transcriptions, feature types, parent names from `mv_pn_srch`.
- **`spelling_fts`** — All 245K written forms from `spelling`.
- **`notes_fts`** — Topics and full text from `snote`.

### Scripts

- **`mysql2sqlite.py`** — Converts the upstream MySQL dump to SQLite. Has a hand-written SQLite schema (the `SCHEMA` constant) mirroring all MySQL tables, then parses INSERT statements with MySQL→SQLite syntax conversion (backtick removal, escape handling, hex→NULL for geometry). Creates indexes, views, and FTS tables after import.
- **`query_tgaz.py`** — CLI query tool. Name search uses `mv_pn_srch` with LIKE prefix matching, falling back to FTS. `--history` mode does multi-table joins to assemble full place records.
- **`harvest_tgaz.py`** — Alternative API harvester (not needed if MySQL dump is available). Uses recursive prefix subdivision to work around the API's 200-result cap.

### Key Patterns

- `sys_id` is the universal place identifier (format: `hvd_NNNNN` for CHGIS, `TBRC_GNNNN` for Tibetan data).
- Coordinates are stored as TEXT in `placename` (`x_coord`=longitude, `y_coord`=latitude). Views cast to REAL.
- Years use negative integers for BCE (e.g. `-202` = 202 BCE). Value `9999` means "still existing."
- `tgaz.db` is tracked via Git LFS (228 MB).
