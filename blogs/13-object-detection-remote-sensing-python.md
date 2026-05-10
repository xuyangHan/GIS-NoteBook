# Object Detection on Remote Sensing Imagery

This post is about detecting discrete objects in aerial and satellite imagery using Python-based GeoAI workflows.

The goal is to understand how object detection moves from image chips and labels to geospatial outputs that can be mapped, queried, and evaluated.

Learning goals:

1. Understand what object detection means for remote sensing imagery.
2. Recognize the data preparation steps behind detection models.
3. Avoid common geospatial mistakes in training and validation.
4. Convert model detections into useful GIS outputs.

Earlier posts that pair well with this one:

- [Imagery ETL and Distributed Geospatial Processing](08-imagery-etl-and-distributed-geospatial-processing.md)
- [Raster, Spatiotemporal Data, and Computer Vision in Python](11-raster-spatiotemporal-computer-vision-python.md)
- [PostGIS, Geospatial APIs, and GeoAI Systems in Python](12-postgis-apis-geoai-python.md)

---

## 1) What object detection means

Object detection finds instances of things in imagery.

A detector usually predicts:

- class label,
- bounding box,
- confidence score.

In remote sensing, common targets include:

- buildings,
- vehicles,
- ships,
- trees,
- solar panels,
- damaged structures,
- aircraft,
- storage tanks.

Core idea:

- **Object detection is for discrete objects with countable instances, not continuous surfaces.**

Interview one-liner:

"In remote sensing, object detection turns image pixels into geospatial feature candidates with class labels, boxes, and confidence scores."

---

## 2) Detection vs classification vs segmentation

Image classification answers:

- "What is in this tile?"

Object detection answers:

- "Where are the objects, and what are they?"

Semantic segmentation answers:

- "What class is each pixel?"

Use detection when:

- the target is countable,
- approximate location is enough,
- bounding boxes are acceptable,
- objects are visually separable.

Use segmentation when:

- boundaries matter,
- area matters,
- the output should be a class raster or footprint-like geometry.

---

## 3) Remote sensing detection workflow

A practical workflow usually looks like this:

1. Source imagery with known CRS, resolution, date, and band information.
2. Create image chips or tiles.
3. Collect or convert labels into boxes.
4. Split data by region, scene, or date.
5. Normalize bands and apply augmentation.
6. Train a detector.
7. Run inference over target imagery.
8. Merge overlapping tile predictions.
9. Convert pixel boxes to map coordinates.
10. Publish detections as GeoJSON, GeoPackage, PostGIS, or tiles.

PyTorch is often used for custom training loops and research-style experimentation.

TorchGeo can help with geospatial datasets, sampling, and transforms.

Ultralytics or YOLO-style models are common practical examples for bounding-box detection.

Core idea:

- **The model sees image chips, but the final product must still behave like spatial data.**

---

## 4) Chips, tiles, and labels

Detection models usually train on fixed-size image chips.

Each chip needs:

- image array,
- chip bounds,
- transform from pixel to map coordinates,
- label boxes in chip pixel coordinates,
- class IDs,
- nodata or mask handling,
- source scene metadata.

Labels may come from:

- hand annotation,
- existing vector data,
- authoritative inventories,
- previous model outputs reviewed by humans.

When vector labels are converted to boxes:

- reproject labels to the imagery CRS,
- clip labels to each chip,
- drop or mark labels that are mostly outside the chip,
- keep a link to the original feature ID when possible.

---

## 5) Spatial resolution and object size

Resolution controls what can be detected.

A vehicle may be visible in very high-resolution aerial imagery but not in coarse satellite imagery. A building may be detectable at one resolution but too small or blurry at another.

Questions to ask:

- How many pixels wide is the target object?
- Does the imagery date match the label date?
- Are shadows, clouds, snow, or seasonal changes hiding objects?
- Are objects visually distinct from the background?

Core idea:

- **If the object is not visually represented at the imagery resolution, model choice will not fix the problem.**

---

## 6) Train, validation, and test splits

Random patch splits are risky in geospatial ML.

Nearby chips often share:

- buildings,
- roads,
- land patterns,
- acquisition conditions,
- label style.

If nearby chips appear in both training and validation, performance may look better than it really is.

Better split strategies:

- hold out complete regions,
- hold out complete source scenes,
- split by acquisition date,
- test on a different city or landscape type.

Core idea:

- **Validation should test generalization to new places or scenes, not memorization of nearby pixels.**

---

## 7) Inference and merging predictions

Large imagery is usually processed tile by tile.

Inference often needs:

- overlapping tiles,
- edge padding,
- batching,
- non-maximum suppression,
- confidence thresholds,
- class-specific filtering.

Overlap helps detect objects near tile edges, but it can create duplicate predictions.

Post-processing should:

- merge duplicate boxes,
- remove boxes outside valid imagery,
- attach confidence and model version,
- preserve source imagery metadata,
- keep enough information for review.

---

## 8) From pixel boxes to GIS features

A model predicts boxes in pixel coordinates.

GIS users need map coordinates.

Conversion requires:

- chip origin,
- raster transform,
- CRS,
- resolution,
- any scaling applied during preprocessing.

Common outputs:

- GeoJSON for lightweight exchange,
- GeoPackage for desktop GIS use,
- PostGIS table for querying and review,
- vector tiles for map display,
- CSV plus geometry when downstream tools expect tables.

Useful attributes:

- class name,
- confidence,
- model version,
- imagery source,
- acquisition date,
- inference date,
- review status.

Core idea:

- **A detection is not operationally useful until it has location, confidence, provenance, and a review path.**

---

## 9) Common anti-patterns

- Training on random chips from the same small area and calling it generalization.
- Ignoring imagery resolution relative to object size.
- Treating labels from a different date as perfect truth.
- Dropping CRS and transform during chip generation.
- Forgetting to merge duplicate predictions from overlapping tiles.
- Publishing boxes without confidence scores.
- Comparing counts without checking false positives and false negatives.
- Assuming bounding boxes are accurate footprints.
- Using cloudy, shadowed, or nodata pixels as normal training examples.

---

## 10) Interview-ready Q&A

### Q1: What is object detection in remote sensing?

It is a computer vision task that identifies object instances in imagery and predicts class labels, bounding boxes, and confidence scores.

### Q2: Why are random patch splits risky?

They can leak nearby spatial context between training and validation, making model performance look better than it will be on new places.

### Q3: Why use overlapping tiles during inference?

Overlap reduces missed detections near tile edges, but duplicate predictions must be merged afterward.

### Q4: What makes detection output geospatial?

Predicted pixel boxes must be converted back to map coordinates using the chip location, raster transform, and CRS.

### Q5: When is segmentation better than detection?

Segmentation is better when exact boundaries, area, or pixel-level class maps matter.

### Q6: What metadata should a detection layer include?

Class, confidence, CRS, source imagery, acquisition date, model version, inference date, and review status.

---

## 11) 30-minute mini exercise

Design a vehicle detection workflow from aerial imagery to a map-ready PostGIS layer:

1. Source imagery and confirm CRS, resolution, date, and extent.
2. Create training chips and bounding-box labels.
3. Split data by neighborhood or source scene.
4. Train a detector using a PyTorch or YOLO-style workflow.
5. Run inference with overlapping tiles.
6. Merge duplicate detections.
7. Convert pixel boxes to geospatial polygons.
8. Store results in PostGIS with confidence and model metadata.
9. Define how a human reviewer would accept or reject detections.

Write the checks that would prevent publishing bad detections.

---

## 12) One-page recap

- Object detection finds countable object instances.
- Remote sensing detections usually use image chips and box labels.
- PyTorch, TorchGeo, and YOLO-style tools are common practical options.
- Resolution controls whether objects are visible enough to detect.
- Spatial splits are safer than random patch splits.
- Overlapping inference tiles need duplicate removal.
- Pixel boxes must be converted back to map coordinates.
- Useful GIS outputs include confidence, provenance, and review metadata.
