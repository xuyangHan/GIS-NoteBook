# GIS Storage, Performance, and Delivery in Plain Language

This post is about making GIS workflows fast and usable at scale.  
Goal: help you understand why queries become slow, how indexing fixes that, and how map tiles/raster overviews make maps load smoothly.

This combines the next four planned topics into one practical guide:

1. Spatial indexing fundamentals (R-tree, quad-tree intuition)
2. Query patterns and indexing impact on performance
3. Map tiles and imagery tiling systems (XYZ/TMS, pyramid levels)
4. Raster pyramids, overviews, and rendering/performance trade-offs

---

## 1) Why GIS performance gets slow

At small scale, almost any GIS workflow feels fine.  
At real scale (millions of features, large rasters, many users), slowdowns appear quickly.

Common bottlenecks:
- scanning every feature for every query,
- computing heavy geometry operations too early,
- moving too much data over network,
- rendering full-resolution rasters at every zoom level.

Core idea:
- **Performance comes from reducing work** (fewer candidates, cheaper checks, smarter data delivery).

---

## 2) Spatial indexing: the idea before the details

A spatial index is a shortcut that avoids checking every geometry.

Without index:
- "Find parcels near this road" might check every parcel.

With index:
- first find only nearby candidate parcels quickly,
- then run exact geometry checks on that smaller set.

Think of it as a map-aware lookup table that narrows search space.

---

## 3) R-tree and quad-tree intuition (no heavy theory)

## R-tree (very common in spatial databases)

R-tree groups nearby objects into nested bounding boxes.

When you query an area:
- boxes that do not intersect query area are skipped fast,
- only intersecting boxes are explored deeper.

Why it works:
- fewer disk reads and geometry comparisons.

Good for:
- general vector datasets,
- dynamic or uneven feature distributions.

## Quad-tree (space split into four recursively)

Quad-tree divides space into 4 cells, then splits busy cells again.

Why it works:
- easy space partitioning,
- good for tiled or grid-like use cases.

Good for:
- some tiling workflows,
- point-heavy datasets,
- hierarchical map display logic.

Interview-ready difference:
- R-tree groups by data distribution (object-driven),
- quad-tree splits by space structure (space-driven).

---

## 4) Query patterns that matter most in GIS

Not all spatial queries behave the same.

High-frequency patterns:
- bounding box search (`bbox`)
- intersects/within/contains
- nearest neighbor (`kNN`)
- spatial join (point-in-polygon, polygon overlap)

Performance usually follows this pattern:
1. cheap filter first (index + bbox),
2. exact geometry predicate second.

If you jump directly to exact geometry checks on full tables, performance drops fast.

---

## 5) Practical indexing strategy (what to do in real projects)

### Rule 1: Index geometry columns by default

If a table is used for spatial filters/joins repeatedly, index it.

### Rule 2: Combine spatial and attribute filtering

Spatial index narrows location candidates; attribute filters reduce them further.

Example:
- "Roads intersecting flood zone AND class = primary"

### Rule 3: Filter broad -> narrow

Use coarse checks first, then expensive operations.

Typical flow:
- bbox prefilter,
- exact relationship (`intersects`, `within`),
- optional post-processing.

### Rule 4: Measure, do not guess

Use query plans/timing to verify that indexes are actually used.

---

## 6) Why index exists but query is still slow

Common causes:

1. Query pattern bypasses index-friendly path  
2. Functions/casts prevent optimizer from using index  
3. Massive candidate set still returned (index helps but data is huge)  
4. Missing simplification/generalization for map display  
5. Too many joins at once without staged filtering

Takeaway:
- Indexing is necessary, not always sufficient.
- Query structure and data design still matter.

---

## 7) Map tiles: why web maps feel fast

Loading one giant map image is slow and inflexible.  
Tiles solve this by splitting maps into small pieces.

Core concepts:
- map is divided into fixed-size tiles (often 256x256 or 512x512),
- organized by zoom levels in a pyramid,
- client requests only visible tiles.

Result:
- faster initial load,
- smooth pan/zoom,
- scalable caching and CDN delivery.

---

## 8) XYZ vs TMS (what interviewers expect you to know)

Both are tile addressing schemes with row/column/zoom, but row direction differs.

### XYZ
- common in modern web mapping,
- y index grows downward from top-left origin.

### TMS
- older convention in many systems,
- y index direction is effectively flipped relative to XYZ.

Practical issue:
- if scheme mismatch is not handled, tiles appear inverted/misaligned.

Quick interview line:
"Same zoom/x logic, different y-axis convention. Always verify tile scheme when integrating services."

---

## 9) Tile pyramid levels (zoom intuition)

At each zoom level:
- map is split into more tiles,
- visual detail increases,
- request volume can increase.

Design trade-off:
- more zoom levels = better detail, bigger storage/build cost.

That is why production maps balance:
- max zoom needs,
- storage budget,
- generation time,
- expected user workflows.

---

## 10) Raster pyramids and overviews (big performance win)

Large rasters are expensive to render at full resolution for every zoom.

**Overviews/pyramids** are precomputed lower-resolution versions of a raster.

When zoomed out:
- renderer uses lower-resolution overview instead of full raster.

Benefits:
- much faster draw time,
- lower CPU and I/O cost,
- better user experience.

Trade-offs:
- extra storage,
- preprocessing time,
- need refresh if source raster changes.

For real products, this is usually worth it.

---

## 11) Rendering and delivery trade-offs (vector, raster, tiles)

### Vector delivery strengths
- crisp styling at multiple zooms,
- interactive feature-level behavior.

### Raster delivery strengths
- great for imagery and pre-rendered cartography,
- simple client rendering.

### Tile strategy strengths
- scalable distribution,
- cache-friendly,
- predictable performance.

Good systems often mix:
- vector for interactive thematic layers,
- raster tiles for heavy imagery basemaps.

---

## 12) Performance playbook you can apply immediately

1. Add spatial index on active geometry tables  
2. Rewrite queries to do bbox/index prefilter first  
3. Profile top slow queries and check plan usage  
4. Generate tiles for high-traffic map layers  
5. Build raster overviews for large imagery  
6. Cache aggressively (service layer + CDN where relevant)

This sequence gives quick wins without huge architecture changes.

---

## 13) Interview-ready Q&A (high-frequency)

### Q1: What does a spatial index do?
It narrows candidate features quickly using spatial structure, so exact geometry checks run on a much smaller subset.

### Q2: R-tree vs quad-tree in one sentence?
R-tree groups objects by bounding boxes based on data distribution, while quad-tree recursively partitions space into fixed quadrants.

### Q3: Why are spatial joins slow?
They can trigger many geometry comparisons; performance improves by indexing both sides and using coarse filters before exact predicates.

### Q4: Why use tiles for web maps?
Tiles request only visible map pieces, which improves load time, cacheability, and interactive performance.

### Q5: XYZ vs TMS difference?
They differ mainly in y-axis tile indexing convention; mismatch leads to flipped or misaligned tiles.

### Q6: What are raster overviews and why use them?
Precomputed lower-resolution raster layers used at smaller scales to reduce rendering cost and speed up map display.

---

## 14) 20-minute mini exercise (performance thinking)

Scenario: A city dashboard has:
- 8 million parcel polygons,
- road network,
- flood zones,
- high-resolution imagery background.

Users complain:
- map loads slowly,
- spatial filters lag,
- zooming imagery is choppy.

Do this:

1. Propose one indexing improvement and explain why.  
2. Rewrite one query approach in "coarse filter -> exact check" order.  
3. Decide if parcels should be vector tiles, raster tiles, or direct features for this dashboard.  
4. Explain where overviews should be used and why.  
5. Give a 60-second "performance fix plan" interview answer.

If you can explain trade-offs clearly, you are already stronger than many candidates.

---

## 15) Common anti-patterns to avoid

1. Assuming indexes solve every performance issue without query redesign  
2. Running exact geometry predicates on full datasets first  
3. Serving very large rasters without pyramids/overviews  
4. Mixing tile schemes (XYZ/TMS) without explicit handling  
5. Optimizing only backend queries and ignoring tile/network delivery

---

## 16) One-page recap

- GIS performance is mostly about reducing unnecessary work.
- Spatial indexes accelerate candidate filtering before expensive geometry checks.
- R-tree and quad-tree use different partitioning ideas, both useful in context.
- Query order matters: coarse filter first, exact predicate second.
- Tiles make web maps scalable and responsive.
- XYZ/TMS mismatch can break map alignment quickly.
- Raster overviews are a high-impact way to improve rendering speed.

---

## 17) What to learn next

After this post, the best next step is:
- end-to-end GIS design walkthroughs,
- debugging scenarios across CRS + indexing + tiling,
- interview case drills with structured answers.

That completes the path from foundational concepts to interview-ready system reasoning.
