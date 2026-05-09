# Geospatial Databases and PostGIS

This post is for anyone who needs to **store, query, and run GIS in production** using a normal database.  
We use **PostgreSQL + PostGIS** as the reference stack because it is widely used in jobs and interviews.

This single guide combines three planned topics into one practical path:

1. PostGIS in one mental model (geometry vs geography, CRS in the DB, basic predicates)
2. Indexes, the query planner, and fast spatial queries
3. Production workflows: schema, backups, replication, partitioning, and when one server is not enough

Earlier posts that pair well with this one:

- [GIS Reference Systems and Transformations](03-gis-reference-systems-and-transformations.md) for CRS and reprojection logic.
- [GIS Storage, Performance, and Delivery](04-gis-storage-performance-and-delivery.md) for spatial indexing intuition and “coarse filter first” thinking.

---

## 1) Why put GIS in a database at all?

Files (shapefiles, GeoJSON, GeoPackage) are fine for one-off work.  
A database shines when you need:

- **many concurrent readers** (apps, dashboards, APIs),
- **consistent updates** (transactions, constraints),
- **joins** between spatial and non-spatial tables,
- **indexes** so “find nearby / inside / intersecting” stays fast as data grows.

Interview one-liner:  
“A spatial database is where geometry lives next to attributes, with indexes and SQL so you can query and update at scale.”

---

## 2) PostGIS in one mental model

PostGIS adds a **geometry type** (and related types) to PostgreSQL so rows can store shapes: points, lines, polygons, collections.

You still think in tables:

- one row = one feature (often),
- columns = attributes + a geometry column,
- SQL = filter, join, aggregate like any app database.

### Geometry vs geography (the decision people trip on)

- **geometry**  
  - Stored on a **flat, projected plane** (or sometimes lat/lon stored as “geometry” with a geographic CRS—still computed as Cartesian unless you use geography-specific functions).  
  - **Fast** for most local and regional work.  
  - Best when your CRS is chosen for the job (meters on the ground).

- **geography**  
  - Treats coordinates on a **spherical model** of the Earth.  
  - **More expensive** for many operations.  
  - Handy when you truly need **global** distance/area thinking in degrees without picking a single projection.

Practical rule for interviews:

- **Local project in a known area?** Prefer **geometry** in a **suitable projected CRS** (meters).  
- **Global “within 5 km” without caring about a local projection?** **geography** can be appropriate (with the cost trade-off).

If you are unsure, say: “I’d pick geometry with a well-chosen SRID for regional analysis; I’d use geography when the use case is global and projection choice is awkward.”

### SRID and transforms inside the database

- **SRID** (spatial reference ID) tags which CRS the coordinates use (e.g. WGS 84 geographic vs a UTM zone).
- **ST_Transform(geom, target_srid)** reprojects coordinates in the database.

Why it matters:

- Mixing unknown or wrong SRIDs causes wrong answers (same story as misaligned layers in desktop GIS).
- Buffering in **degrees** is not the same as buffering in **meters**.

Cross-check decisions with the CRS post above; the database does not remove CRS responsibility.

### Storing vector features

Typical pattern:

- `id`, business keys, attributes…
- `geom geometry(Point, 4326)` or `geometry(MultiPolygon, 3857)` etc.

You can enforce type and SRID in the column definition so bad data fails early.

### Raster in PostGIS (high level only)

PostGIS can store **raster** tiles in-database. That is powerful for some workflows (analysis close to vectors) but **not** always the best place for huge cloud imagery catalogs (often object storage + COG + metadata is cheaper and simpler).

Interview line:  
“I’d use PostGIS raster when the database is the natural home for the analysis; for massive imagery libraries I’d often keep rasters outside and reference them, unless there is a strong query pattern that benefits from in-DB raster.”

### Basic predicates you should know cold

These appear in apps and interviews:

- **ST_Intersects(A, B)** — overlaps in any way.
- **ST_Within(A, B)** — A is fully inside B.
- **ST_Contains(A, B)** — A contains B.
- **ST_DWithin(A, B, distance)** — within a distance **in CRS units** (meters only if CRS is projected in meters!).
- **ST_Distance** — distance in CRS units (watch geography vs geometry).
- **ST_Area**, **ST_Length**, **ST_Buffer** — measure and derive.

Always state: **distance and area mean what the CRS allows them to mean.**

### When is the database the right serving layer?

Good fit:

- feature APIs backed by filters and attributes,
- internal tools querying live data,
- moderate tile or bbox-driven reads with caching in front.

Weaker alone for:

- **planet-scale basemap raster tiles** without a CDN/tile pyramid strategy,
- **heavy batch ETL** that might be cheaper on object storage + a compute engine.

Strong interview answer pattern:

1. **Who reads and how often?**  
2. **Freshness vs cache?**  
3. **Query pattern** (bbox, id, attribute, spatial join)?  
4. **Scale path** (read replicas, connection pooling, partitioning)?

---

## 3) Indexes, planners, and fast spatial queries

PostGIS leans on **PostgreSQL’s planner**: it decides scan vs index, join order, and so on.

### Spatial indexes (intuition)

The common choice is **GiST** on geometry (R-tree-ish behavior: bounding boxes rule out huge parts of the table fast).

**SP-GiST** exists for some use cases but **GiST** is the default mental model.

This matches the “index narrows candidates, then exact geometry runs” pattern from [GIS Storage, Performance, and Delivery](04-gis-storage-performance-and-delivery.md).

### The bbox prefilter pattern

Many spatial operations **internally compare bounding boxes first**, then refine. Your job is to write queries so the planner can **use the index** for that first step.

Friendly mental model:

1. Cheap: “could these boxes overlap?”  
2. Expensive: exact intersection test on full geometry.

### Clustering (optional but real)

**Clustering** table storage by a spatial index order (one-time or occasional `CLUSTER`) can improve sequential reads for map-like bbox queries on large tables. It is not magic; it is layout optimization.

### VACUUM and ANALYZE

- **VACUUM** keeps the table healthy (visibility, bloat management).  
- **ANALYZE** updates **statistics** so the planner picks reasonable plans.

After big loads, **ANALYZE** (or autovacuum doing its job) matters: bad stats → wrong plan → slow spatial queries.

### Reading EXPLAIN (no PhD required)

Run `EXPLAIN (ANALYZE, BUFFERS)` on slow queries in a dev environment.

Look for:

- **Seq Scan** on a huge table when you expected an **Index Scan** / **Bitmap Index Scan** on the geometry index,
- **high row counts** early in the plan,
- **nested loop** joining huge sets without selective filters.

If the plan ignores the spatial index, your WHERE clause may be preventing index use.

### Anti-pattern: wrapping the geometry column in a function

Example of what hurts:

- `WHERE ST_Transform(geom, 3857) && something` on every row without a matching expression index.

If you transform in the filter on the **column side**, the planner often cannot use a plain GiST index on `geom`.

Safer patterns (conceptually):

- store data in the CRS you query most, or  
- transform the **parameter** side (the search shape) to match the table SRID, or  
- add a **generated column** / **functional index** if you truly need a repeated transform (trade-off: storage and maintenance).

Interview sound bite:  
“I avoid applying non-immutable transforms to the indexed column in the WHERE clause unless I add a matching index or restructure the query.”

### Attribute filters + spatial filters

Best performance often combines:

- **indexed attribute** filter (country, tenant_id, date range),
- **spatial index** filter (bbox or `&&`),

so both reduce rows before heavy geometry work.

---

## 4) Production PostGIS workflows

### Schema design and migrations

Treat spatial tables like production app tables:

- **primary keys**, foreign keys where it makes sense,
- **constraints** (valid geometry, correct SRID),
- **migrations** (repeatable schema changes, not manual click-ops only).

PostGIS can enforce **valid** geometries; invalid polygons break expectations in overlays and area calculations.

### Backups and restore

- Use normal PostgreSQL backup strategy (logical or physical).  
- **Point-in-time recovery** matters for production.  
- Test restore: a backup you never restore is a wish.

Spatial gotcha: huge tables and large indexes mean **restore time** and **disk** need planning.

### Replication and read scaling

- **Streaming replication** to read replicas is a common pattern for read-heavy map/API traffic.  
- Writes still go to the primary; replicas may lag slightly (know your freshness SLAs).

Interview: “I’d separate read traffic to replicas when queries are heavy and slightly stale reads are OK.”

### Partitioning (region, time, tenant)

When a single table becomes huge:

- **Partition** by **time** (ingest date) or **region** (admin boundary, coarse grid) or **tenant** if multi-tenant.

Benefits:

- smaller indexes per partition,
- easier maintenance (detach old partitions),
- queries that include the partition key prune partitions automatically.

Spatial note: partitioning key should align with **how people query**. Random partitions with no query filter do not help.

### Versioning and auditing patterns

Production systems often need:

- **who changed what** (audit table, triggers),
- **soft deletes** vs hard deletes,
- sometimes **history** tables for legal or operational traceability.

GIS-specific: bulk imports from external agencies may require **job ids** and **source lineage** columns.

### When one node is not enough (honest scale-out talk)

PostGIS scales **up** well for many workloads (bigger machine, better disks, tuned memory).  
**Out** scaling writes is harder than read scaling.

Reasonable story in interviews:

- **Read scale:** replicas, caching (Redis), tile caches, CDN for static tiles.  
- **Write scale:** split datasets, partition, pipeline bulk loads off-peak, sometimes move heavy analytics to a warehouse or batch engine.  
- **Extreme scale:** specialist stacks (but do not name-drop without explaining the **why**).

---

## 5) Common mistakes (short list)

1. **Wrong SRID** or undefined CRS metadata — silent wrong distances.  
2. **Buffering in degrees** and calling it meters.  
3. **No spatial index** on large geometry tables.  
4. **Transform-on-column** in filters without a plan — index bypass.  
5. **Ignoring ANALYZE** after big loads — bad plans.  
6. **Storing global imagery** in-database because “PostGIS can” — cost surprise.  
7. **Invalid geometries** in production — mysterious overlay failures.

---

## 6) Interview-ready Q&A

### Q1: What is PostGIS?

An extension to PostgreSQL that adds spatial types, functions, and indexes so you can store and query geographic data with SQL.

### Q2: geometry vs geography?

Geometry is typically planar math (fast, CRS-dependent). Geography uses a spherical model (more global, often slower). For regional work, geometry in a good projected CRS is common.

### Q3: What is an SRID?

It identifies the coordinate reference system for stored coordinates; transforms use it to reproject between systems.

### Q4: How do spatial indexes speed queries?

They narrow candidate rows using bounding-box approximations before expensive exact tests.

### Q5: Why might `EXPLAIN` show a sequential scan?

Missing index, non-selective filter, function on indexed column preventing index use, or stale statistics.

### Q6: How would you scale read-heavy map queries?

Connection pooling, spatial and attribute indexes, read replicas, caching frequently requested tiles or query results, and CDN where appropriate.

### Q7: When would you not use PostGIS as the main imagery store?

When data volume, range-request COG workflows, or catalog metadata are better served by object storage plus compute—using the DB for vectors and references.

---

## 7) 25-minute mini exercise (paper or SQL)

Pick a city-scale use case: **parcels** + **buildings** + **zoning polygons**.

1. Choose **geometry vs geography** and justify.  
2. Pick an **SRID strategy** for storage vs analysis.  
3. Write (on paper) two queries:  
   - parcels **intersecting** a polygon,  
   - buildings **within 200 m** of a point (check units!).  
4. List **two indexes** you would create and why.  
5. Name one **production** concern (backups, replicas, partitioning) you would plan for at 10× data growth.

---

## 8) One-page recap

- PostGIS puts **features in tables** with **geometry** (+ optional geography) and **SRID**.
- CRS still matters; the database inherits all CRS pitfalls.
- **ST_Intersects**, **ST_DWithin**, buffers, area, length are core building blocks—watch **units**.
- **GiST** on geometry is your default spatial speed tool; bbox prefilter is the mental model.
- Help the planner: selective filters, **avoid blind function-wrapping** of indexed columns, keep **statistics** fresh.
- Production means **constraints, backups, replicas, partitioning**, and a realistic **scale story** beyond “buy bigger iron.”

---

## 9) What to read next

Natural follow-ups from the geo data backend roadmap:

- cloud **ingestion and validation pipelines**,
- **STAC / COG** style imagery catalogs,
- **distributed** batch processing when the database stops being the cheap place for heavy raster crunching.

Those topics connect this database foundation to full modern geospatial data platforms.
