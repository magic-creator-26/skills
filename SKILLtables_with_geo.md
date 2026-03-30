---
name: enterprise-table-analysis
description: >
  Use this skill whenever you have one or more enterprise data tables to analyze in order to answer
  a user query, partially or fully. Triggers include: queries about customers, lawyers, employees,
  transactions, payments, meetings, lawsuits, parking records, or any organizational event data.
  Also use when the answer may require reasoning across multiple tables, inferring implicit events,
  or combining tabular data with documents. Use even when only one table is available — this skill
  governs the full analysis workflow from ingestion to structured output.
---

# Enterprise Table Analysis Skill

This skill governs how to analyze enterprise data tables — individually and jointly — to answer
user queries. Tables arrive pre-filtered by entity and wrapped in Pydantic models. Your job is
**not** to filter — it is to reason, manipulate, fuse, and report.

The LLM executing this skill is a mid-tier model. **Follow every step explicitly. Do not skip
steps. Do not try to do multiple steps at once.**

---

## Step 0: Read What You Have

Before doing anything else, enumerate your inputs.

For each table, extract and write down:
- `table.name`
- `table.description` (treat as a weak hint, not ground truth)
- Column names and their declared ontology types (if any). Ontology-tagged columns are
  authoritative: a column tagged `NationalID` is a national ID even if named `col_7`.
- Row count and a 3–5 row sample

Also note:
- What documents (if any) were retrieved alongside the tables
- The user's query, in your own words (one sentence)

**Do not proceed until you have written this inventory.**

---

## Step 1: Understand Each Table's True Purpose

The `description` field is unreliable. Infer the true purpose of each table from evidence:

1. **Ontology tags first.** If columns are tagged (`Phone`, `Name`, `NationalID`, `AccountID`,
   etc.), use those to identify what kind of entity the table tracks. These are authoritative.
2. **Column names second.** Look for patterns: date/time columns suggest transactional data;
   name + ID columns suggest reference/entity data.
3. **Value samples + internal entity extractor third.** For columns with no ontology tag and an
   ambiguous name (e.g. `col_3`, `ref`, `key`, `value`), take 3–5 sample cell values and call
   the internal entity extractor to resolve their type:

   ```python
   import requests

   def extract_entities(text: str) -> dict:
       response = requests.post(
           "http://localhost:8080/extract_identifiers",
           json={"text": text},
           timeout=10
       )
       response.raise_for_status()
       return response.json()

   # Probe an untagged column
   sample_text = " ".join(str(v) for v in df["ref"].dropna().head(5).tolist())
   entities = extract_entities(sample_text)
   # If result contains {"NationalID": [...]} → treat this column as NationalID
   ```

   If the API returns a consistent ontology type for the sampled values, treat that column
   as carrying that type — same authority as an explicit tag.
   If the API fails (timeout, connection error), fall back to visual inspection and mark the
   column as `type_uncertain` in your notes.

4. **Cross-reference with other tables.** If a column in table A shares values with a column in
   table B (especially an ontology-tagged or API-resolved column), that is a join key and tells
   you something about both tables' purpose.
5. **Cross-reference with documents.** If a retrieved document mentions names or IDs that appear
   in a table, that table is likely relevant to the document's subject. IDs resolved by the
   entity extractor in documents can be matched directly against API-resolved table columns.

For each table, write one sentence: *"This table appears to record [X] about [Y]."*

---

## Step 2: Classify Tables by Type

Label each table as one of:

- **Reference** — describes an entity (customer profile, employee record, partner info)
- **Transactional** — records events over time (payments, meetings, expenses, parking)
- **Relational** — links two entity types (lawyer–case assignments, customer–product mappings)
- **Geo-enriched** — contains spatial data as a primary or significant column: coordinates,
  addresses, zone codes, district IDs, or geometry objects. May overlap with the above types
  (e.g. a customer table can be both Reference and Geo-enriched).
- **Geo-reference** — a pure spatial lookup table: branch locations, district boundaries,
  zone polygons, address-to-coordinate mappings. No transactional or entity rows of its own.
- **Uncertain** — you cannot determine the type yet; revisit after Step 4

---

## Step 2b: Detect and Prepare Geometric Columns

Run this step only if any table was classified as Geo-enriched or Geo-reference. Skip otherwise.

### Identify geometric columns

Look for columns that contain any of the following:
- Decimal coordinate pairs (`lat`/`lon`, `x`/`y`, `latitude`/`longitude`)
- ITM (Israeli Transverse Mercator) coordinate pairs — large integers, typically
  x ≈ 100,000–300,000 and y ≈ 400,000–800,000
- WKT strings (`POINT(...)`, `POLYGON(...)`, `LINESTRING(...)`)
- Address strings (street + city, or structured address fields)
- Zone codes, district IDs, or region labels that map to a spatial reference table

Write down: column name, suspected type, sample values, and suspected CRS.

### Resolve the Coordinate Reference System (CRS)

This is critical — mixing CRS without converting will produce completely wrong spatial results.

| Signal | Likely CRS |
|---|---|
| Values like `(32.08, 34.78)` | WGS84 (EPSG:4326) — standard lat/lon |
| Large integers like `(179500, 665000)` | ITM (EPSG:2039) — Israeli grid |
| Column explicitly tagged `ITM` or `TM` | ITM (EPSG:2039) |
| Column tagged `WGS84` or `GPS` | WGS84 (EPSG:4326) |
| Unknown | Assume WGS84 unless values look like ITM integers |

**If mixing tables with different CRS, always reproject to a common CRS before any spatial
operation.** Use ITM (EPSG:2039) for Israel-domain queries (better metric accuracy);
use WGS84 for anything requiring standard lat/lon output.

### Build the GeoDataFrame

```python
import geopandas as gpd
from shapely.geometry import Point

# From separate lat/lon columns (WGS84)
gdf = gpd.GeoDataFrame(
    df,
    geometry=gpd.points_from_xy(df["longitude"], df["latitude"]),
    crs="EPSG:4326"
)

# From ITM coordinate columns
gdf = gpd.GeoDataFrame(
    df,
    geometry=gpd.points_from_xy(df["x_itm"], df["y_itm"]),
    crs="EPSG:2039"
)

# Reproject ITM → WGS84 if needed
gdf = gdf.to_crs("EPSG:4326")

# Reproject WGS84 → ITM for distance calculations in meters
gdf = gdf.to_crs("EPSG:2039")
```

If the table contains WKT geometry strings:
```python
from shapely import wkt
df["geometry"] = df["geom_col"].apply(wkt.loads)
gdf = gpd.GeoDataFrame(df, geometry="geometry", crs="EPSG:4326")
```

If the table contains only address strings (no coordinates), note this as a
**geocoding requirement** in your plan. Do not attempt geocoding unless an explicit
geocoding tool or API is available. Flag the gap in `missing_data` output.

---

## Step 3: Plan Required Manipulations

Look at the user's query. For each table (or pair of tables), decide what manipulation is needed.
Write the plan before executing it.

Common manipulation types:

| Need | Operation |
|---|---|
| Count events | Aggregation: `groupby` + `count` |
| Sum or average a value | Aggregation: `groupby` + `sum` / `mean` |
| Link two tables | Join on shared ID or ontology-matched column |
| Isolate a time range | Filter on date column |
| Find co-occurrences | Merge + filter |
| Remove duplicate rows | Deduplication: `drop_duplicates` |
| Compute elapsed time | Derived column: timestamp difference |
| Find the latest event | Sort + `head(1)` or `idxmax` |
| Find nearby entities | Spatial: `gdf.geometry.distance(point) <= radius` (use ITM for meters) |
| Find entities within a zone/polygon | Spatial: `gpd.sjoin(gdf, zones, predicate="within")` |
| Cluster by location | Spatial: group by zone code, or use `gdf.dissolve` on a region column |
| Find closest branch/office to each entity | Spatial: `gpd.sjoin_nearest(gdf_entities, gdf_branches)` |
| Check if two events happened in same location | Spatial: distance between points < threshold |
| Spatial join two geo tables | `gpd.sjoin(gdf_a, gdf_b, how="left", predicate="intersects")` |

**Always reproject to ITM (EPSG:2039) before any distance or area calculation** — distances in
WGS84 degrees are not meaningful in meters. Reproject back to WGS84 only for final output if
needed.

**Use pandas for most non-spatial operations.** Use polars only if you have very large tables
(>500k rows) and need performance. Do not mix the two in one pipeline.
**Use geopandas for any spatial operation** — do not attempt to compute distances or spatial
joins manually from raw coordinate columns.

Write out your manipulation plan like this:
```
Table "payments": filter by customer_id = X, aggregate by month, count rows
Table "meetings": filter by lawyer_id = Y and customer_id = X, count rows
Join "expenses" to "meetings" on date + lawyer_id to find unlogged meetings
```

---

## Step 4: Execute Manipulations — One Table at a Time

Execute the plan from Step 3. For each table:

1. Convert the Pydantic table to a DataFrame:
   ```python
   df = pd.DataFrame(table.rows)  # or table.to_dataframe() if the model exposes that
   ```
2. Apply the planned operations step by step. Do not chain more than 2–3 operations
   without an intermediate result you can inspect.
3. After each operation, note: how many rows remain, what the key columns look like.
4. If a result is empty or unexpected, note it explicitly — do not silently skip it.

---

## Step 5: Cross-Table Reasoning — Find Implicit Evidence

This is the most important step. **An event that is not recorded in the expected table may still
be provable from another table.**

After individual manipulations are complete, perform cross-table evidence fusion:

### 5a. Find Join Keys
For every pair of tables, check:
- Do any ontology-tagged columns match across tables? (`NationalID` in table A and table B → same person)
- Do any column names match or are semantically similar? (`lawyer_id`, `atty_id`, `rep_id` may all be the same thing)
- Do any value sets overlap? Sample 10 values from each and check

Write down all join keys you find.

### 5b. Look for Implicit Events
Ask yourself: *"Is there evidence in table B that an event happened, even though it is not recorded in table A?"*

Common inference patterns:

| Implicit event | Evidence to look for |
|---|---|
| Unlogged meeting between A and B | Expense record showing A billed on same date + location as B |
| Payment not in payment table | Bank/cash record or invoice with matching amount and date |
| Presence at a location | Parking record, badge swipe, or travel expense |
| A relationship exists | Two entities co-appear in a transactional record (e.g. same case, same invoice) |
| Two people were at the same place | Both have location records within spatial threshold on the same date |
| An event occurred at a specific branch/office | Entity's location record falls within branch polygon on event date |
| A person's address falls in a specific district | Address/coordinates fall inside district boundary (spatial join) |

For each inferred event:
- State what the inference is
- State which table(s) provide the evidence
- Rate your confidence: **HIGH** (strong corroboration), **MEDIUM** (plausible but indirect), **LOW** (speculative)

### 5c. Check Documents
If documents were retrieved, check whether any document corroborates or contradicts findings
from the tables. Note any match.

---

## Step 6: Detect Contradictions

Before producing output, scan for contradictions:
- Does table A say an event happened on date X while table B implies date Y?
- Does a reference table say a person has role R, but a transaction table shows behavior inconsistent with R?
- Are there duplicate records with conflicting values?

For each contradiction:
- Describe it clearly
- Do **not** silently resolve it by picking one side
- Flag it in the output as a contradiction for the downstream synthesis model to handle

---

## Step 7: Produce Structured Evidence Output

Output the following structure. Fill in every field. If a field is unknown, write `null` and
explain why.

```json
{
  "query_interpretation": "One sentence restatement of what the user is asking",

  "tables_analyzed": [
    {
      "name": "table name",
      "inferred_purpose": "what this table records",
      "manipulation_applied": "what you did to it",
      "key_finding": "what you found"
    }
  ],

  "direct_findings": [
    {
      "finding": "fact derived directly from table data",
      "source_tables": ["table_a", "table_b"],
      "confidence": "HIGH | MEDIUM | LOW",
      "detail": "supporting numbers or values"
    }
  ],

  "inferred_findings": [
    {
      "finding": "fact inferred across tables or from implicit evidence",
      "reasoning": "step-by-step explanation of the inference",
      "source_tables": ["table_a", "table_b"],
      "confidence": "HIGH | MEDIUM | LOW"
    }
  ],

  "contradictions": [
    {
      "description": "what conflicts",
      "sources": ["table_a", "table_b"],
      "unresolved": true
    }
  ],

  "document_corroborations": [
    {
      "document": "document identifier or title",
      "corroborates": "which finding it supports or contradicts"
    }
  ],

  "geo_findings": [
    {
      "finding": "spatially-derived fact (proximity, containment, co-location)",
      "operation": "distance | within_zone | spatial_join | co-location",
      "crs_used": "EPSG:2039 | EPSG:4326",
      "source_tables": ["table_a", "table_b"],
      "confidence": "HIGH | MEDIUM | LOW",
      "detail": "e.g. distance = 340m, zone = 'Tel Aviv North', threshold used"
    }
  ],

  "missing_data": [
    {
      "what_is_missing": "description of gap",
      "impact_on_answer": "how this limits the answer"
    }
  ],

  "answer_summary": "2–4 sentence plain-language summary of the answer, including confidence caveats"
}
```

---

## Confidence Calibration Guide

Use this to assign confidence ratings honestly:

| Level | Meaning |
|---|---|
| HIGH | Directly observed in table data; no inference required; no contradicting evidence |
| MEDIUM | Requires one inference step OR supported by two indirect signals |
| LOW | Requires multiple inference steps, or evidence is a single indirect signal, or contradicting evidence exists |

When in doubt, go one level lower. Do not inflate confidence to make the answer look cleaner.

---

## Common Failure Modes to Avoid

- **Treating description as ground truth.** Always verify against the data.
- **Stopping at the expected table.** If the meeting log is empty, look elsewhere before concluding "no meetings."
- **Ignoring ontology tags.** A column tagged `NationalID` is your most reliable join key — use it.
- **Silent empty results.** If a filter returns 0 rows, say so explicitly. It may be meaningful.
- **Merging contradictions.** If two sources disagree, report both. Do not average or pick.
- **Over-confident inference.** Every extra inference step lowers confidence. Reflect that in the rating.
- **Computing distances in WGS84.** Always reproject to ITM before distance calculations. A degree of longitude near Tel Aviv is ~90km — raw degree differences are meaningless as distances.
- **Assuming CRS without checking.** Large integers are likely ITM; small decimals are likely WGS84. Verify before building the GeoDataFrame.
- **Treating address strings as coordinates.** If a column contains free-text addresses and no coordinate columns are present, flag it as a geocoding gap — do not attempt to parse lat/lon out of address strings manually.
