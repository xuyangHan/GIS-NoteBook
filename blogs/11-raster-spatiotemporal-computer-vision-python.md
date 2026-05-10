# Raster, Spatiotemporal Data, and Computer Vision in Python

This post is about Python workflows for raster data, gridded time series, and image-based spatial analysis.

The goal is to connect three related ideas: rasters are arrays with geography, climate-style datasets are labeled multidimensional arrays, and computer vision can extract visual patterns when spatial metadata is handled carefully.

Learning goals:

1. Understand raster metadata, transforms, CRS, bands, masks, and nodata.
2. Know when to use Rasterio, NumPy, Xarray, and OpenCV.
3. Recognize raster, spatiotemporal, and image-processing workflow shapes.
4. Avoid losing geospatial meaning while doing array and image operations.

Earlier posts that pair well with this one:

- [GIS Storage, Performance, and Delivery](04-gis-storage-performance-and-delivery.md)
- [Imagery ETL and Distributed Geospatial Processing](08-imagery-etl-and-distributed-geospatial-processing.md)
- [Vector GIS, CRS, and Spatial ETL in Python](10-vector-gis-crs-etl-python.md)

---

## 1) Raster data is pixels plus geography

A raster is a grid of cells.

Geospatial rasters also carry:

- CRS,
- affine transform,
- resolution,
- extent,
- band descriptions,
- nodata values,
- masks,
- data type,
- overviews.

Core idea:

- **The array values are only half the raster. The metadata tells you where those pixels are on Earth.**

Interview one-liner:

"Rasterio lets Python read raster values and geospatial metadata together, while NumPy handles efficient array math."

---

## 2) Rasterio and NumPy workflow

Rasterio handles geospatial file logic:

- open GeoTIFFs,
- read windows,
- inspect CRS and transform,
- write output rasters,
- reproject and resample,
- apply masks.

NumPy handles array logic:

- band math,
- thresholds,
- statistics,
- classification rules,
- local calculations.

The workflow is usually:

1. Read raster data and metadata with Rasterio.
2. Process values with NumPy.
3. Write results with updated Rasterio metadata.

High-frequency raster operations:

- clip raster by boundary,
- read a window instead of a whole file,
- reproject raster to another CRS,
- resample to another resolution,
- mask nodata values,
- calculate indices such as NDVI,
- derive slope or simple terrain products,
- convert raster classes to vector polygons when needed.

---

## 3) Spatiotemporal data with Xarray

Some data is not just one raster.

Examples:

- daily temperature grids,
- monthly precipitation,
- ocean salinity by depth and time,
- climate model outputs,
- satellite time series,
- forecast ensembles.

These datasets may have dimensions like:

- time,
- latitude,
- longitude,
- band,
- depth,
- scenario,
- ensemble member.

Core idea:

- **Xarray is useful when the labels on array dimensions are as important as the values.**

Rasterio feels like "open this raster file."

Xarray feels like "analyze this labeled data cube."

---

## 4) Xarray workflow shape

A climate-style workflow often looks like this:

1. Open a NetCDF, Zarr, or multi-file dataset.
2. Inspect dimensions, coordinates, variables, and units.
3. Select a time range and spatial bounding box.
4. Convert CRS or coordinate conventions if needed.
5. Aggregate by month, season, year, or region.
6. Export summary tables, rasters, or model-ready arrays.

Key checks:

- coordinate order,
- units,
- calendar type,
- missing values,
- chunk sizes,
- whether longitude is `0..360` or `-180..180`.

---

## 5) Computer vision with OpenCV

OpenCV is excellent for pixel and image operations:

- filtering,
- thresholding,
- edge detection,
- contours,
- morphology,
- template matching,
- feature detection.

But OpenCV does not automatically preserve CRS, transform, or geospatial metadata.

Core idea:

- **Use Rasterio or GIS tools for spatial context, and OpenCV for image operations.**

Spatial CV workflow:

1. Read imagery and metadata with Rasterio.
2. Convert bands or arrays into an OpenCV-friendly format.
3. Preprocess image values.
4. Apply thresholding, morphology, edges, contours, or feature extraction.
5. Convert pixel results back to geospatial coordinates if needed.
6. Validate against reference data.
7. Export raster masks, polygons, or summary metrics.

---

## 6) Raster and vector together

Many workflows combine raster and vector data:

- clip imagery to a city boundary,
- summarize elevation by watershed,
- sample raster values at sensor locations,
- create polygons from classified pixels,
- mask imagery using parcel or land-cover polygons.

Before mixing them:

- confirm CRS compatibility,
- check resolution,
- check nodata behavior,
- confirm whether analysis should happen at pixel, polygon, or tile level.

---

## 7) Common anti-patterns

- Reading a huge raster into memory when a window is enough.
- Dropping transform or CRS when writing output.
- Treating nodata pixels as real zero values.
- Resampling categorical classes with bilinear interpolation.
- Loading a full global climate dataset into memory too early.
- Aggregating climate data without checking units and calendars.
- Saving OpenCV results as plain images and losing georeferencing.
- Resizing imagery without updating or tracking spatial transform.

---

## 8) Interview-ready Q&A

### Q1: What makes a geospatial raster different from a normal image?

It has spatial metadata such as CRS, transform, resolution, extent, and nodata rules.

### Q2: What does Rasterio do?

Rasterio reads, writes, windows, masks, reprojects, and preserves metadata for raster datasets.

### Q3: When is Xarray better than Rasterio?

When data has time, variables, levels, ensembles, or other dimensions beyond a single raster grid.

### Q4: What is Zarr useful for?

Zarr stores chunked array data in a cloud-friendly layout, useful for large distributed analysis.

### Q5: Why is OpenCV not enough by itself for GIS?

Because it does not manage CRS, geotransforms, nodata, or geospatial metadata.

### Q6: When is classical CV better than deep learning?

When the visual rule is simple, labeled data is limited, and interpretability or speed matters.

---

## 9) 30-minute mini exercise

Design a workflow to extract potential water bodies from imagery and summarize them by watershed:

1. Read source imagery and metadata.
2. Create a water mask using band math or thresholding.
3. Clean the mask with simple morphology.
4. Preserve CRS, transform, nodata, and resolution.
5. Convert mask regions to polygons if needed.
6. Summarize area by watershed.
7. Write final raster and vector outputs.

List the metadata checks that would prevent a bad output.

---

## 10) One-page recap

- Rasters are arrays with spatial metadata.
- Rasterio handles file and geospatial context.
- NumPy handles array math.
- Xarray is for labeled multidimensional arrays.
- NetCDF and Zarr are common gridded data formats.
- OpenCV is image-first, not GIS-first.
- Pixel outputs must be mapped back to coordinates carefully.
- Nodata, transform, CRS, and resampling choices matter.
