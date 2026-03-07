# TGAZ-SQLite

A portable SQLite conversion of the **Temporal Gazetteer (TGAZ)** — the China Historical GIS placename database maintained by Harvard and Fudan Universities. The full dataset ships as a single 228 MB file with no server required.

## What's in the database

| Content | Count |
|---------|-------|
| Historical placenames | 82,117 |
| Spelling variants (Chinese, pinyin, Tibetan, etc.) | 245,042 |
| Jurisdictional relationships | 83,400 |
| Historical notes | 25,655 |
| Feature types | 1,147 |

**Temporal range**: 763 BCE to 1911 CE (with `9999` for still-existing entities)

**Data sources**:
- China Historical GIS (CHGIS) — 77,769 records
- Tibetan Buddhist Resource Center (TBRC) — 3,587 records
- Historical Gazetteer of Russia — 761 records

**Upstream**: Converted from the MySQL dump in [fccs-dci/containerized_tgaz](https://github.com/fccs-dci/containerized_tgaz). The live API is at [chgis.hudci.org/tgw](https://chgis.hudci.org/tgw/).

## Quick start

No dependencies beyond Python 3 (uses only `sqlite3`, `json`, `re` from the standard library).

```bash
git clone https://github.com/sije/tgaz-sqlite.git
cd tgaz-sqlite

# Note: tgaz.db is tracked with Git LFS — make sure git-lfs is installed
# brew install git-lfs   # macOS
# apt install git-lfs    # Debian/Ubuntu

python3 query_tgaz.py "北京"
```

## CLI usage

```
python3 query_tgaz.py [search] [options]
```

### Search by name

Searches the `name` and `transcription` columns using prefix matching. Works with Chinese characters, pinyin, or Tibetan script.

```bash
$ python3 query_tgaz.py "北京"
sys_id     | name | transcription     | beg_yr | end_yr | ftype_vn | ftype_tr   | ...
-----------+------+-------------------+--------+--------+----------+------------+----
hvd_115045 | 北京兆  | Beijingzhao       | 420    | 444    | 京師       | jing shi   | ...
hvd_111929 | 北京路  | Beijing Lu        | 1138   | 1149   | 路        | lu         | ...
hvd_111962 | 北京路  | Beijing Lu        | 1153   | 1263   | 路        | lu         | ...
hvd_111963 | 北京行省 | Beijing Xingsheng | 1264   | 1264   | 行省       | xing sheng | ...
hvd_111989 | 北京行省 | Beijing Xingsheng | 1271   | 1278   | 行省       | xing sheng | ...
hvd_112045 | 北京行部 | Beijing Xingbu    | 1403   | 1421   | 行部       | xing du    | ...
hvd_112048 | 北京行在 | Beijingxingzai    | 1425   | 1441   | 行在       | xing zai   | ...
(7 results)
```

### Filter by year

Use negative numbers for BCE. This finds places that existed during the given year (i.e., `beg_yr <= year AND end_yr >= year`).

```bash
$ python3 query_tgaz.py "长安" --year -200 --format json
[
  {
    "sys_id": "hvd_70621",
    "name": "长安县",
    "transcription": "Changan Xian",
    "beg_yr": -202,
    "end_yr": 8,
    "obj_type": "POINT",
    "longitude": "108.93719",
    "latitude": "34.31799",
    "ftype_vn": "县",
    "ftype_tr": "xian",
    "parent_sys_id": "hvd_115474",
    "parent_vn": "京兆尹",
    "parent_tr": "Jingzhaoyin",
    "data_src": "CHGIS"
  }
]
```

### Full-text search

FTS5-powered search across names, transcriptions, feature types, and parent jurisdictions. Useful for Tibetan and romanized names where prefix matching won't work.

```bash
$ python3 query_tgaz.py --fts "dgon"
sys_id       | name          | transcription  | beg_yr | end_yr | ftype_vn | ftype_tr | data_src
-------------+---------------+----------------+--------+--------+----------+----------+---------
TBRC_G1578   | དགོན་གསར་དགོན | dgon gsar dgon | 1887   | 9999   | དགོན་པ།  | dgon pa  | TBRC
TBRC_G1672   | དགོན་གསར་དགོན | dgon gsar dgon | 1600   | 9999   | དགོན་པ།  | dgon pa  | TBRC
...
```

### Spatial bounding box

Filter by longitude/latitude rectangle, optionally combined with a year.

```bash
$ python3 query_tgaz.py --bbox 108,34,109,35 --year -200
sys_id     | name | transcription | beg_yr | end_yr | ftype_vn | ftype_tr   | ...
-----------+------+---------------+--------+--------+----------+------------+----
hvd_115474 | 京兆尹  | Jingzhaoyin   | -379   | 556    | 郡        | jun        | ...
hvd_115622 | 内史   | Neishi        | -399   | 207    | 王畿       | wang ji    | ...
hvd_116125 | 右扶风  | Youfufeng     | -300   | 573    | 郡        | jun        | ...
hvd_70421  | 好畤县  | Haozhi Xian   | -221   | 13     | 县        | xian       | ...
hvd_70630  | 杜县   | Du Xian       | -687   | -66    | 县        | xian       | ...
...
(11 results)
```

### Spelling variants

Get all known written forms for a placename, including script type and transliteration system.

```bash
$ python3 query_tgaz.py --spellings hvd_70621
written_form | script              | lang    | trsys  | default_per_type | exonym_lang | attested_by
-------------+---------------------+---------+--------+------------------+-------------+------------
Changan Xian | n/a                 | en      | Pinyin | 1                |             |
长安县          | simplified Chinese  | zh      | n/a    | 1                |             |
長安縣          | traditional Chinese | zh-Hant | n/a    | 0                |             |
(3 results)
```

### Place history

Retrieve the full record for a place: metadata, all spellings, parent jurisdictions over time, predecessors, present-day location, and historical notes.

```bash
$ python3 query_tgaz.py --history hvd_70621 --format json
{
  "place": {
    "sys_id": "hvd_70621",
    "beg_yr": -202,
    "end_yr": 8,
    "ftype_en": "county",
    "ftype_vn": "县",
    "x_coord": "108.93719",
    "y_coord": "34.31799",
    "source_name": "China Historical GIS",
    ...
  },
  "spellings": [
    {"written_form": "Changan Xian", "script": "n/a", "lang": "en", "trsys": "Pinyin"},
    {"written_form": "长安县", "script": "simplified Chinese", "lang": "zh"},
    {"written_form": "長安縣", "script": "traditional Chinese", "lang": "zh-Hant"}
  ],
  "parents": [...],
  "predecessors": [...],
  "present_location": [...],
  "notes": [...]
}
```

### Raw SQL

Run arbitrary SELECT queries directly against the database.

```bash
$ python3 query_tgaz.py --sql "SELECT sys_id, name, transcription, beg_yr, end_yr
    FROM mv_pn_srch
    WHERE ftype_tr = 'fu'
    AND beg_yr <= 750 AND end_yr >= 750
    ORDER BY transcription LIMIT 10"
sys_id     | name | transcription | beg_yr | end_yr
-----------+------+---------------+--------+-------
hvd_115104 | 成都府  | Chengdufu     | 663    | 1259
hvd_88178  | 大兴府  | Daxing Fu     | 0      | 1283
hvd_115217 | 凤翔府  | Fengxiangfu   | -687   | 1911
hvd_111181 | 河南府  | Henanfu       | 742    | 755
hvd_115332 | 河南府  | Henanfu       | 644    | 1234
hvd_115340 | 河中府  | Hezhongfu     | 691    | 1368
hvd_111220 | 京兆府  | Jingzhao Fu   | 713    | 755
hvd_115470 | 京兆府  | Jingzhaofu    | 581    | 1293
hvd_115646 | 南阳府  | Nanyang Fu    | 742    | 1273
hvd_115675 | 平阳府  | Pingyang Fu   | 632    | 1304
(10 results)
```

### Output formats

All commands support `--format table` (default), `--format json`, or `--format jsonl`.

### Database statistics

```bash
$ python3 query_tgaz.py --stats
{
  "total_placenames": 82117,
  "total_spellings": 245042,
  "total_relationships": 83400,
  "year_range": {"earliest": -763, "latest": 9999},
  "sources": [
    {"source": "China Historical GIS", "count": 77769},
    {"source": "Tibetan Buddhist Resource Center", "count": 3587},
    {"source": "Historical Gazetteer of Russia", "count": 761}
  ],
  "feature_types": [
    {"type": "villages and small towns", "count": 48690},
    {"type": "county", "count": 12752},
    ...
  ]
}
```

## Using with SQLite directly

You don't need the CLI tool at all — open the database with any SQLite client:

```bash
sqlite3 tgaz.db
```

### Key tables

| Table | Description |
|-------|-------------|
| `mv_pn_srch` | **Start here.** Pre-joined search view with name, transcription, coordinates, feature type, parent jurisdiction. 82K rows. |
| `placename` | Core records with `sys_id`, coordinates (`x_coord`/`y_coord`), temporal range (`beg_yr`/`end_yr`), feature type FK. |
| `spelling` | All 245K name forms (simplified, traditional, pinyin, Tibetan, etc.) linked via `placename_id`. |
| `part_of` | Jurisdictional parent-child relationships with temporal bounds. |
| `ftype` | Feature type lookup: `name_vn` (Chinese), `name_tr` (romanized), `name_en` (English). |
| `snote` | Historical notes. Linked from `placename.snote_id`. |
| `present_loc` | Present-day location mappings. |
| `prec_by` | Predecessor relationships between places. |

### Useful views

| View | Description |
|------|-------------|
| `v_search` | `mv_pn_srch` + data source names. Best for serializing to LLM context. |
| `v_placename` | `placename` + `data_src` + `ftype` joined. |
| `v_placename_names` | `placename` + all spellings + script + transliteration info. |
| `v_full` | Everything: `placename` + `data_src` + `ftype` + `snote` + `present_loc`. |

### FTS5 full-text indexes

| Index | Content |
|-------|---------|
| `search_fts` | Names, transcriptions, feature types, parent names from `mv_pn_srch` |
| `spelling_fts` | All 245K written forms from `spelling` |
| `notes_fts` | Topics and full text from `snote` |

```sql
-- FTS example: search for places related to 洛阳
SELECT * FROM search_fts WHERE search_fts MATCH '洛阳';

-- Find all spelling forms containing "chang"
SELECT * FROM spelling_fts WHERE spelling_fts MATCH 'chang*';
```

### Example queries

```sql
-- All counties that existed in the year 100 CE
SELECT sys_id, name, transcription, beg_yr, end_yr
FROM mv_pn_srch
WHERE ftype_tr = 'xian' AND beg_yr <= 100 AND end_yr >= 100;

-- Jurisdictional parents of a place over time
SELECT po.beg_yr, po.end_yr, p.sys_id, s.name, s.transcription
FROM part_of po
JOIN placename p ON po.parent_id = p.id
JOIN mv_pn_srch s ON p.sys_id = s.sys_id
WHERE po.child_id = (SELECT id FROM placename WHERE sys_id = 'hvd_70621');

-- Count of counties per dynasty (using representative years)
SELECT
  CASE
    WHEN 221 BETWEEN -beg_yr AND -end_yr THEN 'Qin'
    WHEN beg_yr <= -200 AND end_yr >= -200 THEN 'W. Han'
    WHEN beg_yr <= 100 AND end_yr >= 100 THEN 'E. Han'
    WHEN beg_yr <= 750 AND end_yr >= 750 THEN 'Tang'
    WHEN beg_yr <= 1100 AND end_yr >= 1100 THEN 'N. Song'
    WHEN beg_yr <= 1400 AND end_yr >= 1400 THEN 'Ming'
    WHEN beg_yr <= 1820 AND end_yr >= 1820 THEN 'Qing'
  END AS dynasty,
  COUNT(DISTINCT sys_id) AS county_count
FROM mv_pn_srch
WHERE ftype_tr = 'xian'
GROUP BY dynasty
HAVING dynasty IS NOT NULL;

-- Places within 1 degree of Xi'an
SELECT sys_id, name, transcription,
       CAST(x_coord AS REAL) AS lon, CAST(y_coord AS REAL) AS lat
FROM placename
WHERE CAST(x_coord AS REAL) BETWEEN 107.9 AND 109.9
  AND CAST(y_coord AS REAL) BETWEEN 33.3 AND 35.3
  AND beg_yr <= 750 AND end_yr >= 750;
```

## Data conventions

- **`sys_id`** is the universal identifier. Format: `hvd_NNNNN` (CHGIS), `TBRC_GNNNN` (Tibetan), `rgaz_NNNNNN` (Russian).
- **Coordinates** are stored as TEXT in `placename` (`x_coord` = longitude, `y_coord` = latitude). Use `CAST(x_coord AS REAL)` for numeric comparisons. The views already do this for you.
- **Years** use negative integers for BCE: `-202` = 202 BCE. The value `9999` means "still existing."
- **Feature types** have three name columns: `name_vn` (Chinese/native), `name_tr` (romanized transcription), `name_en` (English).

## Using with LLMs

This database is designed to work well with LLM-powered query tools. The `v_search` view provides a clean, denormalized format suitable for context windows, and the FTS indexes enable fuzzy name matching without exact character knowledge.

A [Claude Code skill](.claude/skills/tgaz-query/SKILL.md) is included that teaches Claude the schema and common query patterns. When used with Claude Code, it reduces token usage by ~21% compared to unaided querying while maintaining the same accuracy.

## Rebuilding from source

To regenerate `tgaz.db` from the upstream MySQL dump:

```bash
git clone https://github.com/fccs-dci/containerized_tgaz.git /tmp/tgaz_repo
python3 mysql2sqlite.py
# Optionally specify paths: python3 mysql2sqlite.py tgaz.db /path/to/dump.sql
```

The conversion takes about 2 seconds. The script uses a hand-written SQLite schema (rather than automated MySQL-to-SQLite conversion) to ensure correctness, then parses INSERT statements with syntax adaptation.

## License

The TGAZ data is from the [China Historical GIS](https://chgis.fairbank.fas.harvard.edu/) project (Harvard & Fudan University). See the upstream repository for data licensing terms. The conversion scripts in this repository are provided as-is.
