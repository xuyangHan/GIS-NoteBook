# Cloud Geospatial Data Pipelines

This post is about how geospatial data moves from **raw files and feeds** into reliable datasets that apps, maps, analysts, and APIs can use.

The goal is not to sell one cloud vendor or one tool.  
The goal is to understand the **pipeline shape** that appears again and again in real GIS teams:

1. Pipeline anatomy: raw -> curated -> served
2. Orchestration, infrastructure, and data quality
3. Serving layers after the warehouse

Earlier posts that pair well with this one:

- [GIS Storage, Performance, and Delivery](04-gis-storage-performance-and-delivery.md) for tiling, caching, and delivery intuition.
- [Geospatial Databases and PostGIS](06-geospatial-databases-postgis.md) for spatial database patterns after data is cleaned.
- [GIS Reference Systems and Transformations](03-gis-reference-systems-and-transformations.md) for CRS and reprojection decisions.

---

## 1) Why geospatial pipelines are different

A normal data pipeline already worries about:

- files arriving late,
- schema changes,
- missing values,
- failed jobs,
- duplicate records,
- downstream consumers depending on stable output.

Geospatial pipelines add extra failure modes:

- wrong or missing **CRS**,
- invalid polygons,
- mixed geometry types,
- unexpected spatial extent,
- very large rasters,
- tile pyramids and cached outputs,
- map services that need low latency, not just correct tables.

Core idea:

- **A geospatial pipeline is not just ETL with coordinates.**  
  It must protect spatial meaning from raw ingest all the way to user-facing delivery.

Interview one-liner:

"A cloud geospatial pipeline should make data repeatable, validated, traceable, and easy to serve through maps, APIs, tiles, or analytics tables."

---

## 2) Pipeline anatomy: raw -> curated -> served

Many teams describe pipeline stages as **bronze / silver / gold**.

You can also say:

- **ingest**
- **validate**
- **publish**

The words matter less than the separation of responsibility.

### Bronze: raw landing zone

Bronze is where data first lands.

Examples:

- agency shapefile zip upload,
- GeoJSON export from a partner system,
- GPS traces from devices,
- satellite imagery files,
- parcel updates from an FTP/SFTP feed,
- event stream with lon/lat points.

Bronze should preserve the source as much as possible.

Typical rule:

- keep original files,
- record when they arrived,
- record where they came from,
- do not overwrite old raw inputs silently.

Why this matters:

- if later processing fails, you can replay,
- if results are questioned, you can trace back to the source,
- if a vendor changes format, you can compare before and after.

### Silver: validated and normalized data

Silver is where raw inputs become trustworthy enough for analysis.

Common steps:

- unzip / parse / load,
- normalize column names,
- assign or transform CRS,
- validate geometry type,
- fix or reject invalid geometries,
- deduplicate records,
- standardize timestamps and IDs,
- check extents against expected area.

This is where most geospatial pipeline mistakes should be caught.

Practical example:

- A city parcel file should not suddenly contain features in another country.
- A road centerline dataset should not arrive as polygons.
- A WGS 84 point feed should not have coordinates like `(623000, 4832000)` unless it is actually projected data mislabeled as lon/lat.

### Gold: publishable datasets

Gold is the version built for consumption.

Examples:

- clean PostGIS tables,
- analytics tables in a warehouse,
- vector tile layers,
- COG/STAC imagery catalogs,
- feature APIs,
- public downloads,
- dashboard-ready aggregates.

Gold should be shaped around use cases:

- map display,
- spatial joins,
- search and filter,
- downloads,
- reporting,
- machine learning features.

The gold layer is not always one dataset.  
A single source may produce:

- a full-resolution internal table,
- a simplified public map layer,
- a vector tile layer,
- a monthly summary table.

---

## 3) Metadata and lineage stubs

A pipeline should always answer:

- Where did this dataset come from?
- When was it processed?
- Which version of the pipeline produced it?
- What checks passed or failed?
- Which output tables/files came from which input?

You do not need a huge metadata platform on day one.  
Even simple lineage columns help:

- `source_name`
- `source_url`
- `source_file`
- `source_updated_at`
- `ingested_at`
- `pipeline_run_id`
- `schema_version`
- `processing_status`

For batch files, store a small manifest:

- file name,
- checksum,
- row/feature count,
- bounding box,
- CRS,
- geometry type,
- processing result.

Interview line:

"I would start with lightweight lineage and manifests, then add a formal catalog when the team needs search, governance, or cross-team discovery."

---

## 4) Validation gates: CRS, schema, geometry

A validation gate is a check that must pass before data moves forward.

### CRS validation

Check:

- CRS exists,
- CRS matches expectation,
- coordinates are plausible,
- transformations happen intentionally.

Common problem:

- A file claims EPSG:4326 but contains projected meter coordinates.

Good pipeline behavior:

- reject it,
- quarantine it,
- alert someone,
- or mark it as needing manual review.

### Schema validation

Check:

- required fields exist,
- field types match expectation,
- important IDs are not null,
- enum/category values are valid,
- dates parse cleanly.

For spatial datasets, a schema contract often includes:

- geometry column name,
- geometry type,
- CRS,
- required attributes,
- primary/business key,
- update frequency,
- expected spatial extent.

### Geometry validation

Check:

- valid geometries,
- expected type (Point/LineString/Polygon/MultiPolygon),
- non-empty geometry,
- no impossible coordinates,
- sensible area/length ranges,
- topology rules where relevant.

Topology examples:

- parcels should not overlap within the same jurisdiction,
- road segments should connect at expected nodes,
- administrative boundaries should not leave unexplained gaps,
- polygons should not self-intersect.

Not every dataset needs every check.  
The check should match the business meaning of the layer.

---

## 5) Failure handling and retries

Cloud pipelines fail for boring reasons:

- network timeout,
- file not ready yet,
- permission change,
- schema drift,
- temporary database lock,
- service rate limit.

They also fail for spatial reasons:

- invalid geometry,
- huge feature unexpectedly breaks memory,
- CRS mismatch,
- raster missing overviews,
- tile generation fails at one zoom.

Good failure handling means:

- retry temporary failures,
- stop on true data quality failures,
- quarantine bad inputs,
- keep enough logs to debug,
- do not publish partial broken output as if it is clean.

Practical rule:

- **retry infrastructure problems, review data meaning problems.**

### Idempotent writes

Idempotent means you can safely run the same pipeline again without duplicating or corrupting output.

Useful patterns:

- write to a temporary/staging table first,
- validate counts and checks,
- swap/rename into production only after success,
- use `pipeline_run_id` or source version keys,
- upsert by stable business key,
- avoid blind append unless append-only history is intentional.

Why it matters:

- retries become safe,
- failed jobs can be replayed,
- scheduled pipelines do not create duplicates every time they rerun.

Interview line:

"I prefer staging writes plus validation, then an atomic publish step so consumers either see the old good version or the new good version, not half a load."

---

## 6) Orchestration: scheduled vs event-driven

Orchestration is how pipeline steps run in the right order.

### Scheduled pipelines

Best when data arrives predictably:

- nightly parcel refresh,
- weekly zoning update,
- monthly imagery index rebuild,
- hourly GPS aggregation.

Strengths:

- simple mental model,
- easy to monitor,
- good for batch systems.

Weaknesses:

- may run when no new data exists,
- may miss off-schedule urgent updates,
- freshness depends on schedule interval.

### Event-driven ingestion

Best when data arrives irregularly or needs fast reaction:

- file lands in object storage,
- webhook says a vendor export is ready,
- message arrives from an IoT/device stream,
- user uploads a dataset.

Strengths:

- reacts quickly,
- avoids unnecessary runs,
- fits streaming or near-real-time workflows.

Weaknesses:

- harder to reason about,
- needs careful retry and dedup logic,
- event storms can overload downstream systems.

Practical answer:

- use scheduled jobs for predictable batch data,
- use event-driven triggers when arrival time matters,
- combine both when needed (event-driven ingest, scheduled quality report).

---

## 7) Infrastructure-as-code, conceptually

Infrastructure-as-code means your cloud resources are defined in version-controlled configuration, not only clicked into existence.

For a geospatial pipeline, that might include:

- storage buckets/containers,
- databases,
- permissions,
- job schedules,
- queues,
- secrets references,
- monitoring alerts,
- networking rules,
- compute jobs.

The point is not the specific tool.  
The point is repeatability.

Benefits:

- dev/test/prod can be similar,
- changes are reviewed,
- disaster recovery is easier,
- new team members can understand the system shape,
- accidental manual drift is easier to spot.

Interview line:

"I would define storage, compute, permissions, and schedules as code so the pipeline can be reviewed, recreated, and promoted across environments."

---

## 8) Data quality checks that matter in GIS

General checks:

- row count,
- null count,
- duplicate IDs,
- required fields,
- value ranges,
- date freshness,
- schema drift.

Spatial checks:

- CRS matches contract,
- bounding box is within expected extent,
- geometry type is expected,
- geometries are valid and non-empty,
- area/length ranges are plausible,
- topology rules pass,
- raster resolution and band count match expectation,
- raster has overviews where needed.

Completeness checks:

- all expected regions are present,
- all expected time periods are present,
- no missing tiles in a tile pyramid,
- no missing imagery scenes in a catalog,
- feature count is within a reasonable change threshold.

Example threshold:

- If parcel count changes by 0.5%, probably fine.
- If parcel count drops by 60% overnight, stop and review.

The best checks are boring, specific, and tied to how the dataset is used.

---

## 9) Alerting and observability

A pipeline is production only when someone knows when it breaks.

Useful signals:

- job success/failure,
- duration,
- retry count,
- input file count,
- output feature count,
- validation errors,
- publish status,
- freshness of the served layer,
- API/tile error rates after publish.

Good alerts are actionable.

Weak alert:

- "Pipeline failed."

Better alert:

- "Parcel refresh failed CRS validation: expected EPSG:26917, received EPSG:4326. Output was not published. Last good version remains active."

For public geodata, freshness can matter as much as uptime.  
A service may be technically online but still wrong if it serves old emergency closures, stale flood boundaries, or outdated imagery.

---

## 10) Serving layers after the warehouse

Cleaning data is only half the system.  
People still need to use it.

Common serving paths:

- database/warehouse table,
- vector tiles,
- feature service / WFS-like endpoint,
- custom query API,
- raster tiles,
- COG access through a tile gateway,
- static downloads.

The right serving layer depends on access pattern.

---

## 11) Vector tiles vs WFS-ish patterns vs query APIs

### Vector tiles

Vector tiles are great for map display.

Best when:

- users pan/zoom maps,
- layers are large,
- styling happens in the browser,
- exact full geometry is not needed at every zoom,
- caching matters.

Trade-off:

- tiles are display-oriented,
- geometry may be simplified,
- feature attributes may be limited,
- not ideal for arbitrary analysis queries.

### WFS-ish feature access

"WFS-ish" means feature-oriented access: give me actual features, often filtered by bbox, attribute, or layer.

Best when:

- users need downloadable features,
- GIS clients need feature records,
- analysts need exact geometry and attributes,
- edits or feature identity matter.

Trade-off:

- large responses can be slow,
- arbitrary filters can stress the database,
- pagination and limits matter.

### Query API

A custom query API is best when the product has specific questions:

- parcels by owner ID,
- nearest assets to a point,
- flood risk for an address,
- zoning summary for a polygon,
- available imagery scenes for a date range.

Best when:

- access patterns are known,
- you want strong auth/rate limits,
- you want to hide database complexity,
- responses are product-shaped, not generic GIS-shaped.

Practical rule:

- **Tiles for display, feature services for GIS access, query APIs for product workflows.**

---

## 12) Raster, COG, and tile gateways

Large raster datasets are often better stored as files in object storage than as giant database rows.

Common pattern:

- store imagery as **Cloud Optimized GeoTIFFs (COGs)**,
- store metadata in a catalog,
- serve map tiles through a tile gateway,
- cache popular tiles.

Why COGs help:

- clients/services can request only the byte ranges needed,
- overviews support lower zoom display,
- object storage can be cheaper and simpler for large imagery.

This connects directly to the tiling and overview ideas from the storage/performance post.

Good mental model:

- database/catalog answers "what imagery exists?"
- COG/object storage answers "where are the pixels?"
- tile gateway answers "what should the map display at this zoom and bbox?"

---

## 13) Caching, CDN, auth, and rate limits

Serving geodata is partly a performance problem and partly a governance problem.

### Caching

Cache when:

- many users request the same tiles,
- data changes slowly,
- the output is expensive to compute,
- public traffic is unpredictable.

Common cache targets:

- vector tiles,
- raster tiles,
- expensive query responses,
- static downloads.

### CDN intuition

A CDN helps move cacheable content closer to users.

Great for:

- public basemaps,
- imagery tiles,
- static vector tile sets,
- open data downloads.

Less useful for:

- highly personalized queries,
- private data with complex permissions,
- constantly changing real-time features.

### Auth and rate limits

Public geodata still needs protection:

- API keys or tokens,
- per-user quotas,
- rate limits,
- download limits,
- attribution/license enforcement,
- abuse monitoring.

Why:

- one heavy client can degrade service for everyone,
- public APIs can be scraped aggressively,
- private layers may mix with public ones,
- cost can grow fast under uncontrolled traffic.

Interview line:

"I would separate public cacheable delivery from authenticated query access, then add rate limits around anything expensive or sensitive."

---

## 14) A practical pipeline design example

Scenario:

- A city receives weekly parcel updates.
- The public map needs fast display.
- Internal analysts need exact geometry.
- The city wants to keep history.

Reasonable design:

1. Land original vendor zip in raw storage.
2. Create a manifest with checksum, timestamp, source name, and file count.
3. Load into a staging table.
4. Validate schema, CRS, geometry type, extent, feature count, and topology rules.
5. Write cleaned parcels to a versioned curated table.
6. Publish latest good version to internal PostGIS tables.
7. Generate simplified vector tiles for the public web map.
8. Keep previous good version active if validation fails.
9. Alert the team with failure reason and source file link.

This is not fancy.  
It is exactly the kind of boring reliability that makes spatial systems feel professional.

---

## 15) Common anti-patterns to avoid

1. Publishing directly from raw inputs without validation.
2. Treating CRS metadata as optional.
3. Appending every rerun and creating duplicate features.
4. Letting partial failed outputs become "latest."
5. Running topology checks that do not match the dataset's actual meaning.
6. Serving huge feature responses when vector tiles would fit the map use case better.
7. Serving imagery without overviews, tiling, or caching.
8. Having alerts that say a job failed but not what stayed safe.
9. Building vendor-specific architecture before understanding access patterns.

---

## 16) Interview-ready Q&A

### Q1: What are bronze, silver, and gold layers?

Bronze stores raw source data, silver stores validated and normalized data, and gold stores publishable datasets shaped for apps, APIs, analytics, or maps.

### Q2: What checks would you add for a spatial pipeline?

CRS, geometry type, valid geometry, expected extent, required attributes, duplicate IDs, completeness, topology rules, and feature count changes.

### Q3: What does idempotent mean in a data pipeline?

The pipeline can run again safely without creating duplicates or corrupting output. Staging tables, stable keys, run IDs, and atomic publish steps help.

### Q4: Scheduled vs event-driven ingestion?

Scheduled is good for predictable batch updates. Event-driven is good when files or messages arrive irregularly and freshness matters.

### Q5: Why use vector tiles instead of direct feature queries?

Vector tiles are optimized for map display, caching, and pan/zoom performance. Direct feature queries are better when users need exact features or analysis-ready records.

### Q6: Where do COGs fit?

COGs fit large raster and imagery workflows where object storage plus range requests, overviews, and tile gateways are more efficient than loading every pixel into a database.

### Q7: How would you prevent a bad update from breaking a public map?

Load into staging, validate, publish only after checks pass, keep the last good version active, and alert the team with a clear failure reason.

---

## 17) 30-minute mini exercise

Scenario: A regional transportation agency publishes:

- road closures every 5 minutes,
- road centerlines monthly,
- aerial imagery yearly,
- public vector map layers,
- internal analyst downloads.

Do this:

1. Decide which datasets should be scheduled and which should be event-driven.
2. Define bronze/silver/gold outputs for one vector layer.
3. List five validation checks for road centerlines.
4. Choose a serving pattern for public display: vector tiles, feature API, or both.
5. Choose a serving pattern for imagery: raw GeoTIFF download, COG + tile gateway, or pre-rendered raster tiles.
6. Explain how you would keep the last good public version safe during a failed refresh.

If you can explain those choices clearly, you are thinking like a geospatial data engineer, not just a tool user.

---

## 18) One-page recap

- Cloud geospatial pipelines move data from **raw** to **validated** to **served**.
- Bronze/silver/gold is a useful mental model, but ingest/validate/publish says the same thing in simpler words.
- Metadata and lineage make outputs explainable and rerunnable.
- CRS, schema, geometry, extent, topology, and completeness checks are core spatial quality gates.
- Idempotent writes and atomic publish steps prevent duplicate and half-broken outputs.
- Scheduled jobs fit predictable batch data; event-driven ingestion fits irregular or freshness-sensitive data.
- Infrastructure-as-code makes cloud resources repeatable and reviewable.
- Serving choices matter: vector tiles for display, feature services for GIS access, query APIs for product workflows.
- Large imagery often fits object storage + COG + tile gateway better than database-first storage.
- Caching, CDN, auth, and rate limits turn clean data into reliable public geodata services.

---

## 19) What to read next

Natural follow-ups from here:

- distributed geospatial processing,
- STAC and COG imagery catalogs,
- real-time spatial event streams,
- cloud cost patterns for geospatial workloads,
- production monitoring for GIS APIs and tile services.

That is the bridge from "I can process spatial data" to "I can operate geospatial data products."
