# Imagery ETL and Distributed Geospatial Processing

This post is about moving satellite and aerial imagery from **raw scenes** into datasets that can be searched, processed, mosaicked, and served reliably.

The goal is not to memorize every raster tool.  
The goal is to understand the workflow shape that shows up in modern imagery systems:

1. Imagery ETL lifecycle: raw scenes -> optimized assets -> cataloged products
2. STAC, catalogs, and cloud-native access
3. Mosaics, QA, provenance, and distributed processing

Earlier posts that pair well with this one:

- [GIS Storage, Performance, and Delivery](04-gis-storage-performance-and-delivery.md) for COGs, tiles, overviews, and map delivery intuition.
- [Cloud Geospatial Data Pipelines](07-cloud-geospatial-data-pipelines.md) for bronze/silver/gold pipeline thinking.
- [GIS Reference Systems and Transformations](03-gis-reference-systems-and-transformations.md) for CRS and reprojection trade-offs.

---

## 1) Why imagery ETL is different

Vector ETL usually moves features: points, lines, polygons, attributes.

Imagery ETL moves pixels, metadata, footprints, masks, time, resolution, and storage layout.

That adds different failure modes:

- huge files,
- mixed resolutions,
- multiple bands,
- no-data regions,
- clouds, shadows, snow, haze,
- different acquisition dates,
- different projections,
- slow reads if files are not layout-optimized,
- expensive mistakes because every scene may be gigabytes.

Core idea:

- **Imagery ETL is not only about converting files.**  
  It is about making pixels discoverable, readable, comparable, and explainable at scale.

Interview one-liner:

"An imagery pipeline should preserve raw scenes, optimize file layout for access, catalog metadata for search, validate spatial/radiometric quality, and publish products with clear lineage."

---

## 2) Imagery ETL lifecycle

A practical imagery pipeline often looks like this:

1. Land raw scenes.
2. Extract and validate metadata.
3. Normalize naming, projection, bands, and no-data rules where needed.
4. Convert to cloud-friendly formats.
5. Build overviews.
6. Register items in a catalog.
7. Generate mosaics, composites, tiles, or analysis-ready products.
8. Publish the latest good output.

The exact steps depend on the source.

Examples:

- daily satellite scenes,
- yearly aerial orthophotos,
- drone survey orthomosaics,
- elevation rasters,
- land-cover classification outputs,
- weather or climate grids.

The pipeline should keep the original input.  
Derived assets should be reproducible from source files and recorded parameters.

Practical rule:

- raw data is evidence,
- optimized data is for access,
- catalog data is for discovery,
- products are for use.

---

## 3) Formats: GeoTIFF, COG, and why layout matters

**GeoTIFF** is a common raster format that stores pixels plus georeferencing metadata.

A normal GeoTIFF can be perfectly valid but still perform badly in cloud workflows.

**Cloud Optimized GeoTIFF (COG)** is a GeoTIFF arranged so clients can efficiently read only the parts they need, usually through HTTP range requests.

Why COG layout matters:

- internal tiles support windowed reads,
- overviews support zoomed-out reads,
- metadata is positioned for quick inspection,
- object storage can serve byte ranges without downloading the whole file.

This connects directly to the raster overview and delivery ideas in [GIS Storage, Performance, and Delivery](04-gis-storage-performance-and-delivery.md).

Mental model:

- plain GeoTIFF: "Here is a raster file."
- COG: "Here is a raster file arranged for remote partial reads."

Good pipeline behavior:

- validate that outputs are actually COG-compliant,
- choose compression intentionally,
- build overviews,
- preserve band descriptions and no-data values,
- avoid rewriting pixels unless the product requires it.

---

## 4) Internal tiling, overviews, and compression

Large rasters should not be read as one giant block.

Useful layout decisions:

- **internal tile size**: often 256, 512, or 1024 pixels depending on access pattern,
- **compression**: reduces storage and network transfer but adds CPU cost,
- **overviews**: lower-resolution copies used for small-scale display,
- **block alignment**: affects how much extra data gets read for a window.

Trade-off:

- smaller tiles can help small reads but increase metadata and request overhead,
- larger tiles can help scan-style processing but waste reads for tiny windows,
- stronger compression saves storage but may slow repeated processing.

There is no universal best setting.

Practical answer:

- optimize layout for the dominant access pattern: map display, pixel sampling, tile serving, or batch analysis.

---

## 5) Reprojection and resampling trade-offs

Imagery pipelines often need reprojection, but reprojection is not free.

It can:

- move pixels,
- change resolution,
- introduce interpolation artifacts,
- blur sharp features,
- shift category boundaries,
- create edge no-data areas,
- increase processing cost.

Common resampling choices:

- **nearest neighbor** for categorical rasters like land cover or masks,
- **bilinear** for continuous imagery where smoothness is acceptable,
- **cubic** for smoother visual output with more computation,
- **average/mode** when reducing resolution depending on data meaning.

Practical rule:

- continuous values can often tolerate interpolation,
- categorical labels usually need nearest or mode,
- never resample without knowing what the pixel values mean.

Interview line:

"I avoid unnecessary reprojection in raw storage, but I may create analysis-ready or display-ready derivatives in a common CRS when that simplifies downstream use."

---

## 6) Chunking by scene, tile, or grid

Imagery is too large to think of as one file forever.

Common chunking strategies:

- **by scene**: good when source products are natural units,
- **by tile**: good for map display and repeated spatial access,
- **by fixed grid**: good for batch processing and distributed jobs,
- **by time window**: useful for composites and temporal analytics.

Scene-based processing is often embarrassingly parallel:

- process each satellite scene,
- validate metadata,
- convert to COG,
- register catalog item,
- publish result.

Tile or grid-based processing is often better when the output is a seamless product:

- national mosaic,
- land-cover map,
- cloud-free seasonal composite,
- model inference over a large region.

The chunk boundary should match the job.

Bad sign:

- every processing step needs to read the entire archive to produce one small output.

---

## 7) STAC, catalogs, and discovery

Once imagery grows beyond a few folders, discovery becomes a product problem.

People need to ask:

- What imagery exists for this area?
- What date range is available?
- What sensor collected it?
- What cloud cover is acceptable?
- Which file contains the pixels?
- Which bands are available?
- Is this raw, corrected, composited, or derived?

**STAC** means SpatioTemporal Asset Catalog.

High-level model:

- **Collection**: a group of related items, like a satellite mission or aerial survey.
- **Item**: one observation or product with geometry, datetime, properties, and assets.
- **Asset**: a file linked from the item, such as a COG band, thumbnail, metadata file, or mask.

Core idea:

- the catalog answers "what exists and where?"
- object storage answers "where are the bytes?"
- processing engines answer "what should we compute?"

---

## 8) When catalogs replace flat folders

A folder of GeoTIFFs works at small scale.

It breaks when users need:

- search by bbox,
- search by date,
- search by cloud cover,
- search by sensor,
- version filtering,
- product type filtering,
- provenance,
- permissions,
- API access.

Catalogs replace "I know the path" with "I can discover the data."

Flat folder thinking:

- `imagery/2024/scene_123.tif`

Catalog thinking:

- find all items intersecting this bbox,
- between June and August,
- cloud cover below 20%,
- with red, green, blue, NIR, and QA assets,
- from the latest collection version.

Interview line:

"A catalog becomes necessary when path naming is no longer enough to support discovery, filtering, governance, and automation."

---

## 9) Cloud-native access

Cloud-native imagery workflows usually avoid downloading whole files.

Instead they use:

- object storage,
- COGs,
- HTTP range requests,
- metadata catalogs,
- lazy reads,
- chunk-aware processing.

Example:

- a tile server receives a map request,
- searches the catalog for matching imagery,
- reads only the needed COG windows,
- uses overviews when zoomed out,
- returns a rendered tile.

This pattern keeps storage cheap and compute flexible, but it needs good layout.

If files lack internal tiling and overviews, cloud-native access turns into slow remote file dragging.

---

## 10) Mosaics and compositing

A **mosaic** combines multiple images into a larger spatial product.

A **composite** usually combines multiple images over time or quality conditions to choose better pixels.

Examples:

- stitch aerial tiles into a countywide orthomosaic,
- build a cloud-free monthly satellite composite,
- merge scenes from overlapping satellite paths,
- generate a global annual land-cover product.

Common choices:

- newest pixel wins,
- lowest cloud score wins,
- median value across time,
- best observation per season,
- feather/blend seams for display,
- priority by source/sensor/resolution.

The right method depends on product meaning.

For visual basemaps, appearance matters.  
For analysis products, consistency and documented rules matter more than looking pretty.

---

## 11) QA masks: clouds, shadows, snow, and no-data

Imagery often comes with quality information.

Common masks:

- cloud,
- cloud shadow,
- snow,
- water,
- saturated pixels,
- sensor artifacts,
- no-data.

At a high level, masks help decide which pixels should be trusted.

Pipeline checks:

- mask files exist,
- mask resolution aligns with imagery,
- mask classes are documented,
- no-data values are consistent,
- invalid pixels are excluded from composites,
- statistics report how much valid data remains.

Good interview answer:

"For a cloud-free mosaic, I would use QA masks and cloud scores to exclude bad pixels, then report coverage and remaining gaps so the mosaic quality is measurable."

---

## 12) Provenance and version drift

Imagery products drift when:

- source scenes are reprocessed,
- atmospheric correction changes,
- cloud masks improve,
- DEM corrections change,
- pipeline code changes,
- resampling settings change,
- catalog metadata is updated after the fact.

This is why lineage matters.

Track:

- source item IDs,
- source asset URLs,
- source checksums or versions,
- processing date,
- pipeline version,
- parameters,
- CRS,
- resampling method,
- output checksum,
- validation summary.

Without this, "same area, same date" may not actually mean same pixels.

Practical rule:

- if an output could be used in science, insurance, planning, emergency response, or ML training, document what made it.

---

## 13) How to validate a global mosaic

This is a common interview/system-design question.

You cannot inspect every pixel manually.

Use layered validation:

- confirm expected tile count,
- confirm global/regional coverage,
- check no missing tile IDs,
- check CRS, resolution, bounds, and no-data values,
- compare histograms/statistics against expected ranges,
- sample visual chips across biomes and continents,
- measure cloud/no-data percentage by region,
- compare against trusted reference areas,
- verify seam behavior at tile boundaries,
- reproduce a small region from source scenes,
- keep previous version comparison metrics.

Strong answer:

"I would validate structure first, then statistics, then targeted visual samples, then compare against reference areas and the previous product version. The goal is to catch missing data, spatial shifts, bad masks, and unexpected distribution changes before publish."

---

## 14) Distributed geospatial processing: why it is hard

Distributed processing sounds simple:

- split work,
- run in parallel,
- combine results.

Geospatial processing adds friction:

- neighboring pixels or polygons may interact,
- spatial joins can require data movement,
- reprojection can change chunk boundaries,
- large rasters stress network I/O,
- skewed geography creates uneven partitions,
- output must still align spatially.

Core idea:

- **Parallelism works best when chunks can be processed independently.**  
  It gets expensive when the job requires global shuffling or many cross-boundary checks.

---

## 15) Parallelism models that actually work for geo

Good parallel patterns:

- process each imagery scene independently,
- process each tile independently with overlap buffers,
- process each H3/quadkey/grid cell independently,
- run model inference per chip/tile,
- compute zonal statistics per region group,
- pre-aggregate events by spatial cell and time.

Harder patterns:

- global polygon overlay,
- all-to-all spatial joins,
- dissolve across many boundaries,
- nearest-neighbor search without partition pruning,
- repeated reprojection during joins.

Partition choices:

- **H3**: useful for hexagonal indexing and global-ish aggregation,
- **quadkey/Web Mercator tile**: natural for map display and tile workflows,
- **bbox/grid**: simple and flexible for batch jobs,
- **admin region**: useful when outputs are reported by jurisdiction.

Practical rule:

- choose partitions based on the dominant query or output shape, not just because the index sounds modern.

---

## 16) Why shuffle-heavy spatial joins hurt

A distributed join is cheap when each worker can handle its own partition.

A spatial join often hurts because features near partition boundaries may need to move or duplicate.

Example:

- joining billions of points to polygons,
- points are partitioned one way,
- polygons overlap many partitions,
- the engine has to shuffle data so matching candidates meet.

Ways to reduce pain:

- pre-filter by bbox,
- partition both datasets by the same grid,
- duplicate boundary features intentionally,
- simplify geometry for coarse filtering,
- pre-aggregate points by cell before joining,
- use broadcast joins only when one side is small,
- avoid repeated global joins in every pipeline step.

Interview line:

"Spatial joins become expensive when they require large shuffles. I try to co-partition, pre-filter, pre-aggregate, or broadcast the smaller side when that matches the data shape."

---

## 17) Engine landscape: a decision lens

Do not start with a fanboy tool list.

Start with the data shape and access pattern.

### GDAL-oriented batch

Good fit:

- format conversion,
- reprojection,
- COG creation,
- overviews,
- scene-level processing,
- command-line repeatability.

Weak fit alone:

- large distributed joins,
- interactive multi-user analytics,
- complex orchestration without surrounding pipeline tools.

### Dataframe plus geospatial libraries

Good fit:

- large tabular/spatial analytics,
- partitioned processing,
- feature engineering,
- batch joins with known patterns.

Watch:

- geometry serialization cost,
- skew,
- shuffle volume,
- CRS consistency,
- memory pressure.

### Geospatial SQL on big engines

Good fit:

- analysts already use SQL,
- spatial aggregations,
- joins with warehouse data,
- repeatable reporting,
- managed scaling.

Watch:

- function support varies,
- cost can grow with scanned data,
- raster support may be limited or indirect.

### Tile-based map-reduce

Good fit:

- imagery processing,
- raster inference,
- map tile generation,
- grid-aligned products.

Watch:

- edge effects,
- overlap buffers,
- stitching outputs,
- consistent tile schema.

Decision rule:

- pick by data shape, join pattern, output format, and operational team skill.

---

## 18) Cost, latency, and operational reality

Distributed does not automatically mean cheaper.

Common cost drivers:

- reading too much data,
- writing too many intermediate files,
- cross-region egress,
- repeated reprojection,
- unbounded retries,
- large shuffles,
- idle clusters,
- cache misses,
- cold starts.

Latency depends on:

- startup time,
- task scheduling overhead,
- file count,
- object storage request volume,
- network transfer,
- CPU-heavy compression or resampling,
- whether data is already partitioned correctly.

Practical rule:

- optimize I/O before buying more compute.

For imagery, the cheapest byte is usually the byte you never read.

---

## 19) Autoscaling, spot compute, and ingest spikes

Autoscaling helps when jobs are parallel and queue-backed.

It struggles when:

- startup is slow,
- tasks are too tiny,
- one huge task dominates,
- downstream storage cannot handle writes,
- object storage request limits are hit,
- a spatial join creates a shuffle bottleneck.

Spot or preemptible compute can work well for retryable batch jobs.

Good fit:

- COG conversion per scene,
- tile generation,
- model inference per chip,
- independent validation tasks.

Poor fit:

- long non-checkpointed jobs,
- fragile global joins,
- workflows with expensive partial progress loss.

Backpressure matters when ingest spikes.

Use:

- queues,
- concurrency limits,
- retry policies,
- dead-letter/quarantine paths,
- publish gates,
- freshness monitoring.

The system should slow down gracefully instead of publishing half-processed imagery.

---

## 20) Monitoring imagery jobs

Useful signals:

- input scene count,
- output asset count,
- failed scene IDs,
- bytes read and written,
- processing duration by step,
- valid pixel percentage,
- cloud/no-data percentage,
- missing tiles,
- COG validation result,
- catalog registration success,
- mosaic coverage,
- cost per run,
- freshness of latest published product.

Good alert:

- "Daily Sentinel mosaic is 93% complete; 41 scenes failed cloud-mask lookup; previous mosaic remains published."

Weak alert:

- "Raster job failed."

Operational maturity is mostly clarity:

- what failed,
- what was published,
- what stayed safe,
- what needs human review.

---

## 21) A practical design example

Scenario:

- A team receives daily satellite scenes.
- Analysts need search by date and area.
- A web app needs a recent cloud-reduced mosaic.
- ML workflows need reproducible training chips.

Reasonable design:

1. Land original scenes in raw object storage.
2. Extract metadata, footprint, datetime, sensor, band list, and cloud score.
3. Validate CRS, band count, no-data values, bounds, and checksums.
4. Convert imagery bands to COGs with overviews.
5. Register STAC items and assets.
6. Build daily or weekly composites using QA masks.
7. Write mosaic outputs as COGs or tiles depending on serving need.
8. Generate ML chips from versioned catalog queries.
9. Store lineage from source item IDs to each mosaic/chip output.
10. Publish only after coverage and QA thresholds pass.

This design separates:

- storage from discovery,
- discovery from processing,
- processing from publishing.

That separation is what keeps the system understandable as imagery volume grows.

---

## 22) Common anti-patterns to avoid

1. Treating folders of files as a catalog forever.
2. Serving large imagery without COG layout or overviews.
3. Reprojecting every raw scene without a clear downstream reason.
4. Using bilinear/cubic resampling on categorical masks.
5. Building mosaics without documenting pixel selection rules.
6. Publishing cloud-free products without coverage or no-data metrics.
7. Running global spatial joins without partition or shuffle strategy.
8. Scaling compute before reducing reads, writes, and shuffles.
9. Losing lineage between source scenes and derived products.
10. Assuming "latest" means "validated and safe to publish."

---

## 23) Interview-ready Q&A

### Q1: Why use COGs for imagery?

COGs allow efficient partial reads from object storage using internal tiling, overviews, and range requests, so systems do not need to download entire rasters for small windows or map tiles.

### Q2: What is STAC?

STAC is a catalog model for spatiotemporal assets. Collections group related datasets, items describe individual observations or products, and assets link to files such as COGs, masks, thumbnails, or metadata.

### Q3: When would you reproject imagery?

When a product needs a common CRS for analysis, display, or alignment. I would avoid rewriting raw scenes unnecessarily and choose resampling based on whether values are continuous or categorical.

### Q4: How would you build a cloud-free mosaic?

Search catalog items by area and date, use cloud/shadow/snow masks or scores to reject bad pixels, choose a compositing rule, generate the output, then validate coverage, no-data percentage, seams, and sample chips.

### Q5: How do you validate a global mosaic?

Check expected tile coverage, CRS, resolution, no-data values, statistics, regional samples, seam behavior, reference areas, and differences from the previous version before publishing.

### Q6: Why are spatial joins hard in distributed systems?

They often require data movement so matching geometries meet on the same worker. Large shuffles, boundary duplication, and skew can dominate runtime.

### Q7: How do you choose a distributed geo engine?

Start from data shape and join pattern: GDAL-style batch for file transformations, dataframe/geospatial libraries for partitioned analytics, SQL engines for warehouse-style spatial queries, and tile-based map-reduce for raster or map products.

---

## 24) 30-minute mini exercise

Scenario:

- You have 5 years of satellite imagery for a country.
- Product wants a web map of the latest cloud-reduced mosaic.
- Analysts need historical search.
- ML engineers need reproducible training chips.

Do this:

1. Decide what stays in raw storage and what becomes optimized COGs.
2. Define a minimal STAC item: geometry, datetime, properties, and assets.
3. Pick a compositing rule for a monthly cloud-reduced mosaic.
4. List five validation checks before publishing the mosaic.
5. Choose a partitioning strategy for distributed processing.
6. Explain what lineage you would store for each training chip.

If you can explain those decisions clearly, you are thinking like someone who can operate imagery systems, not just open rasters in a desktop GIS.

---

## 25) One-page recap

- Imagery ETL moves pixels, metadata, masks, footprints, time, and provenance.
- GeoTIFF is common; COG is layout-optimized for cloud-native partial reads.
- Internal tiling, compression, and overviews should match access patterns.
- Reprojection and resampling can change pixel meaning, so choose them intentionally.
- STAC catalogs make imagery discoverable by area, time, sensor, quality, and asset type.
- Mosaics and composites need explicit pixel selection rules.
- QA masks help exclude clouds, shadows, snow, no-data, and other bad pixels.
- Provenance prevents confusion when source scenes, masks, or pipeline code change.
- Distributed geo works best when scenes, tiles, or grid cells can process independently.
- Shuffle-heavy spatial joins hurt; co-partition, pre-filter, broadcast, or pre-aggregate when possible.
- Operational reality includes egress, object storage reads, cold starts, retries, monitoring, and publish gates.

---

## 26) What to read next

Natural follow-ups from here:

- real-time geospatial event streams,
- raster machine learning and chip generation,
- vector tile production at scale,
- geospatial lakehouse design,
- production monitoring for maps, catalogs, and imagery APIs.

That is the bridge from "I can process imagery" to "I can design and operate imagery platforms."
