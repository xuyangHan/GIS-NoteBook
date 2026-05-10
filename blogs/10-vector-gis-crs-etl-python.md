# Vector GIS, CRS, and Spatial ETL in Python

This post is about the core Python workflows behind vector GIS: spatial tables, geometry operations, coordinate systems, and repeatable ETL.

The goal is to help you move from one-off scripts to reliable spatial workflows that can survive real data, real CRS problems, and real interview questions.

Learning goals:

1. Understand GeoDataFrames as spatial tables.
2. Use Shapely and GeoPandas for common vector operations.
3. Handle CRS definition, reprojection, distance, and area safely.
4. Design repeatable geospatial ETL checks and outputs.

Earlier posts that pair well with this one:

- [GIS Concepts Foundations](02-gis-concepts-foundations.md)
- [GIS Reference Systems and Transformations](03-gis-reference-systems-and-transformations.md)
- [Python GIS Ecosystem and QGIS Automation](09-python-gis-ecosystem.md)

---

## 1) Vector data in Python

Vector GIS represents geography as:

- points,
- lines,
- polygons,
- attributes.

GeoPandas adds a geometry column to a Pandas-like table. Shapely represents the geometry objects inside that column.

Core idea:

- **A GeoDataFrame is a table plus spatial meaning.** The geometry column, CRS, and attributes must stay consistent.

Interview one-liner:

"GeoPandas is useful when vector data can be treated as a spatial table, and Shapely provides the geometry operations behind that table."

---

## 2) Common vector workflow shape

A practical vector workflow usually looks like this:

1. Read source files or database results.
2. Inspect geometry type, CRS, extent, and columns.
3. Reproject when distance, area, or overlay requires it.
4. Fix invalid geometries if needed.
5. Run spatial operations.
6. Validate row counts, extents, and output fields.
7. Export to GeoPackage, GeoJSON, Parquet, or a database.

Do not skip inspection. Many GIS bugs are not code bugs; they are data meaning bugs.

---

## 3) Operations you should know

High-frequency vector operations:

- **spatial join**: attach attributes based on location.
- **overlay**: create new geometries from intersection, union, difference, or identity.
- **buffer**: create areas around features.
- **dissolve**: merge features by attribute.
- **clip**: keep only geometry inside a boundary.
- **nearest neighbor**: find closest features.
- **validity check**: detect self-intersections or broken polygons.

Practical warning:

- Buffer, area, and distance should usually happen in a projected CRS with meter-like units, not raw latitude/longitude degrees.

---

## 4) CRS in Python: define vs reproject

Coordinates are just numbers until a CRS gives them meaning.

To **define** a CRS means:

- "These existing coordinate numbers should be interpreted as this CRS."

To **reproject** means:

- "Convert these coordinate numbers from one CRS into another CRS."

If coordinates are already longitude/latitude but CRS metadata is missing, define the CRS.

If coordinates are WGS84 and you need meters for distance, reproject to a projected CRS.

Core idea:

- **Do not transform data until you know whether the current CRS metadata is correct.**

---

## 5) Choosing an analysis CRS

Use WGS84 / EPSG:4326 for:

- storage,
- interchange,
- GPS-like coordinates,
- web API inputs.

Use Web Mercator / EPSG:3857 for:

- web map display,
- slippy map tiles.

Use UTM, local state plane, national grids, or equal-area projections for:

- distance,
- area,
- buffering,
- local engineering-style analysis,
- regional statistics.

Fast rule:

- Display CRS is not always analysis CRS.

---

## 6) Geospatial ETL in Python

A geospatial ETL workflow protects spatial meaning while data moves between systems.

Typical stages:

1. Ingest raw data.
2. Validate schema, CRS, geometry, extent, and row counts.
3. Normalize fields, CRS, geometry type, and naming.
4. Transform or enrich data.
5. Write curated outputs.
6. Publish to files, database, tiles, API, or map.

Minimum vector checks:

- expected columns exist,
- geometry column exists,
- CRS is present and expected,
- geometry type is expected,
- geometries are valid,
- extent is plausible,
- row count is plausible,
- required attributes are not null,
- IDs are unique where required.

Core idea:

- **Good ETL makes spatial data repeatable, explainable, and safe to consume downstream.**

---

## 7) Idempotent outputs

An idempotent pipeline can run more than once without corrupting results.

Practical patterns:

- write to a temporary output first,
- validate the temporary output,
- replace the published output only after validation,
- version outputs by date or source batch,
- keep raw inputs unchanged,
- log parameters and source metadata.

This matters when scheduled jobs fail halfway through.

---

## 8) Common anti-patterns

- Running distance calculations in EPSG:4326.
- Assuming all input files have correct CRS metadata.
- Calling `to_crs` on data with incorrect source CRS.
- Assigning EPSG:4326 to projected coordinates just to silence a warning.
- Ignoring invalid polygons before overlay.
- Pulling an entire database table into memory before filtering.
- Overwriting final outputs before validation.
- Logging only "success" without row counts or extents.

---

## 9) Interview-ready Q&A

### Q1: What is a GeoDataFrame?

A GeoDataFrame is a Pandas-like table with a geometry column and CRS metadata.

### Q2: Spatial join vs overlay?

A spatial join transfers attributes based on spatial relationships. Overlay creates new geometries from the combination of input layers.

### Q3: Define CRS vs reproject?

Defining CRS labels existing coordinates. Reprojecting changes coordinate values into another CRS.

### Q4: Why not calculate distance in EPSG:4326?

Because coordinates are angular degrees, not meters, and degree distance changes depending on latitude.

### Q5: What makes geospatial ETL different?

It must preserve CRS, geometry validity, spatial extent, raster metadata when relevant, and output contracts.

### Q6: What does idempotent mean?

It means rerunning the pipeline with the same inputs does not create duplicate, partial, or corrupted outputs.

---

## 10) 30-minute mini exercise

Design a Python ETL workflow for monthly zoning updates:

1. Load new zoning polygons.
2. Check schema, CRS, geometry validity, extent, and row count.
3. Reproject to the project analysis CRS.
4. Repair invalid geometries using a documented policy.
5. Export a curated GeoPackage and a database-ready file.
6. Keep the previous published output if validation fails.

Write the checks, output names, and failure behavior.

---

## 11) One-page recap

- GeoPandas handles spatial tables.
- Shapely handles geometry operations.
- CRS checks come before measurement.
- Define and reproject are different operations.
- Use projected CRS for distance and area.
- ETL is about reliability, not only conversion.
- Keep raw data unchanged.
- Write temporary outputs before publishing.
