# Semantic Segmentation for Land-Use/Land-Cover Mapping

This post is about using semantic segmentation to create land-use and land-cover maps from geospatial imagery.

The goal is to understand how pixel-level model predictions become classified rasters, area summaries, and map-ready GIS products.

Learning goals:

1. Explain semantic segmentation in geospatial terms.
2. Prepare aligned imagery and label masks for training.
3. Evaluate segmentation results with useful metrics.
4. Preserve geospatial meaning when publishing class maps.

Earlier posts that pair well with this one:

- [Raster, Spatiotemporal Data, and Computer Vision in Python](11-raster-spatiotemporal-computer-vision-python.md)
- [PostGIS, Geospatial APIs, and GeoAI Systems in Python](12-postgis-apis-geoai-python.md)
- [Object Detection on Remote Sensing Imagery](13-object-detection-remote-sensing-python.md)

---

## 1) What semantic segmentation means

Semantic segmentation assigns a class label to every pixel.

For land-use and land-cover mapping, classes might include:

- water,
- trees,
- grass,
- crops,
- bare soil,
- buildings,
- roads,
- wetlands,
- snow or ice.

Core idea:

- **Segmentation turns imagery into a categorical raster where every valid pixel has a class.**

Interview one-liner:

"Semantic segmentation is useful for land-cover mapping because the output is a pixel-level class map that can be summarized by area, region, or time."

---

## 2) Segmentation vs detection

Object detection predicts boxes around object instances.

Semantic segmentation predicts a class for each pixel.

Use segmentation when:

- area matters,
- boundaries matter,
- the target is not naturally countable,
- the output should be a raster class map,
- downstream users need statistics by region.

Use detection when:

- objects are countable,
- approximate boxes are enough,
- the output is a candidate inventory.

For example:

- detecting individual vehicles fits object detection,
- mapping all impervious surface pixels fits segmentation.

---

## 3) Land-cover segmentation workflow

A practical workflow usually looks like this:

1. Source imagery and label masks.
2. Confirm imagery and masks align pixel-for-pixel.
3. Define class IDs, class names, and nodata values.
4. Create training chips.
5. Split data by region, scene, or time.
6. Normalize bands and apply augmentation.
7. Train a segmentation model.
8. Evaluate per-class performance.
9. Run inference over target imagery.
10. Mosaic predicted chips into a classified raster.
11. Summarize classes by boundary when needed.
12. Publish GeoTIFFs, polygons, tiles, or database summaries.

PyTorch is commonly used for segmentation training.

TorchGeo can help organize geospatial datasets and sampling.

U-Net-style architectures are common baselines because they work well for dense pixel prediction.

Core idea:

- **The label mask is as important as the image; if they are misaligned, the model learns the wrong lesson.**

---

## 4) Label masks and class IDs

Segmentation labels are often rasters where each pixel stores a class ID.

A label specification should define:

- class ID,
- class name,
- color palette,
- nodata value,
- background policy,
- ignored classes,
- whether classes are mutually exclusive.

Example class policy:

- `0`: nodata or ignore,
- `1`: water,
- `2`: vegetation,
- `3`: built-up,
- `4`: bare soil.

Nodata is not the same as background.

Background means "valid pixel with no target class."

Nodata means "do not train or evaluate here."

---

## 5) Alignment, resolution, and resampling

Imagery and masks must align.

Check:

- CRS,
- transform,
- resolution,
- extent,
- pixel grid origin,
- width and height,
- nodata handling.

If resampling categorical masks:

- use nearest neighbor,
- do not use bilinear or cubic interpolation,
- confirm class IDs are preserved.

If resampling imagery:

- choose a method based on the data type and purpose,
- document the output resolution,
- keep the transform consistent.

Core idea:

- **A beautiful model cannot recover from training data where image pixels and label pixels point to different places.**

---

## 6) Class imbalance and mixed pixels

Land-cover datasets are often imbalanced.

Common examples:

- many vegetation pixels, few wetland pixels,
- many background pixels, few roads,
- many easy rural pixels, fewer dense urban edge cases.

Possible responses:

- sample more chips from rare classes,
- use class weights,
- evaluate per-class metrics,
- review confusion between similar classes,
- collect more examples from underrepresented regions.

Mixed pixels are another challenge.

A single pixel may contain both tree canopy and pavement, especially at coarser resolution.

Core idea:

- **Per-class evaluation matters because a high overall score can hide failure on rare but important classes.**

---

## 7) Evaluation metrics

Useful segmentation metrics include:

- confusion matrix,
- per-class precision,
- per-class recall,
- F1 score,
- Intersection over Union,
- overall accuracy,
- area difference by class.

IoU measures how much predicted class area overlaps reference class area.

For operational products, also inspect:

- maps of false positives,
- maps of false negatives,
- performance by region,
- performance by season,
- performance near class boundaries.

Do not rely only on one global metric.

---

## 8) From prediction to map output

Inference produces predicted class arrays.

To become GIS outputs, predictions need:

- CRS,
- transform,
- resolution,
- class IDs,
- palette or style,
- nodata value,
- model version,
- source imagery metadata.

Common outputs:

- classified GeoTIFF,
- Cloud Optimized GeoTIFF,
- raster tiles for display,
- polygons for selected classes,
- area summary tables by boundary,
- dashboard or API summaries.

Polygonization can help when users need vector features, but it can also create many tiny polygons.

Use cleanup carefully:

- remove tiny artifacts only with a documented rule,
- preserve class IDs,
- avoid simplifying boundaries so much that area statistics become misleading.

---

## 9) Common anti-patterns

- Training with imagery and masks that are not perfectly aligned.
- Treating nodata as a valid class.
- Resampling categorical masks with bilinear interpolation.
- Reporting only overall accuracy.
- Using random patch splits that leak spatial context.
- Ignoring cloud, shadow, snow, or seasonal differences.
- Publishing a class raster without class definitions.
- Polygonizing every class without considering output size.
- Comparing area totals without checking resolution and nodata rules.

---

## 10) Interview-ready Q&A

### Q1: What is semantic segmentation?

Semantic segmentation assigns a class label to every valid pixel in an image.

### Q2: Why is segmentation useful for land-cover mapping?

Because land-cover products often need complete pixel-level class maps and area summaries, not only object counts.

### Q3: Why should categorical masks use nearest-neighbor resampling?

Because interpolation can create invalid class values or blur class boundaries.

### Q4: What is IoU?

Intersection over Union measures the overlap between predicted and reference pixels for a class.

### Q5: Why are per-class metrics important?

They reveal whether the model fails on rare or important classes that may be hidden by strong overall accuracy.

### Q6: What metadata should a classified raster preserve?

CRS, transform, resolution, nodata value, class IDs, class names, source imagery, and model version.

---

## 11) 30-minute mini exercise

Design a land-cover segmentation workflow that produces a classified GeoTIFF and an area summary:

1. Choose classes such as water, vegetation, built-up, and bare soil.
2. Source imagery and matching label masks.
3. Confirm CRS, transform, resolution, extent, and nodata alignment.
4. Create chips and split by region.
5. Train a U-Net-style segmentation model.
6. Evaluate with confusion matrix, F1, and IoU.
7. Run inference over the target area.
8. Mosaic predictions into a georeferenced class raster.
9. Summarize class area by administrative boundary.
10. Publish the raster with class definitions and model metadata.

List the checks that would catch a bad class map before publication.

---

## 12) One-page recap

- Segmentation predicts a class for every pixel.
- Land-cover outputs are usually categorical rasters.
- Image and label masks must align exactly.
- Nodata and background are different concepts.
- Use nearest-neighbor resampling for class masks.
- Per-class metrics matter more than one global score.
- Classified rasters need CRS, transform, class IDs, and nodata rules.
- Polygonization is useful only when vector output serves a real need.
