# Applied Geospatial ML Workflows

This post is about the full shape of applied geospatial machine learning projects, from problem framing to map-ready delivery.

The goal is to move beyond individual model types and understand how data, labels, validation, inference, and outputs fit together in real GeoAI work.

Learning goals:

1. Frame geospatial ML problems around useful outputs.
2. Design reproducible training and inference data pipelines.
3. Choose validation strategies that respect geography and time.
4. Deliver predictions as GIS products people can trust and use.

Earlier posts that pair well with this one:

- [Cloud Geospatial Data Pipelines](07-cloud-geospatial-data-pipelines.md)
- [Imagery ETL and Distributed Geospatial Processing](08-imagery-etl-and-distributed-geospatial-processing.md)
- [Semantic Segmentation for Land-Use/Land-Cover Mapping](14-semantic-segmentation-land-cover-python.md)

---

## 1) Start with the decision

A geospatial ML project should start with the decision or workflow it supports.

Clarify:

- objective,
- prediction unit,
- target variable,
- output audience,
- update frequency,
- acceptable uncertainty,
- delivery format.

Prediction units might be:

- pixels,
- image chips,
- parcels,
- road segments,
- buildings,
- census areas,
- watersheds,
- grid cells.

Core idea:

- **The model task should follow the spatial decision, not the other way around.**

Interview one-liner:

"I frame geospatial ML by defining the output user, prediction unit, validation strategy, and delivery format before choosing a model."

---

## 2) Common geospatial ML task shapes

Different problems need different model shapes.

Image classification:

- classify a tile or scene.

Object detection:

- find countable object instances.

Semantic segmentation:

- classify every pixel.

Change detection:

- compare places across time.

Regression:

- predict continuous values such as biomass, temperature, flood depth, or risk score.

Tabular spatial ML:

- combine spatial joins, raster sampling, distances, and attributes to predict a value for each feature.

Core idea:

- **GeoAI is broader than deep learning on imagery; many strong workflows combine spatial features with standard ML models.**

---

## 3) Data pipeline and provenance

Applied workflows need reproducible data preparation.

A pipeline may include:

- imagery catalogs,
- Cloud Optimized GeoTIFFs,
- STAC metadata,
- vector labels,
- boundary layers,
- raster covariates,
- training chips,
- feature tables,
- model artifacts,
- prediction outputs.

Track:

- source dataset,
- acquisition date,
- processing date,
- CRS,
- resolution,
- label version,
- chip generation parameters,
- split assignment,
- model version.

Core idea:

- **If you cannot recreate the training data, you cannot confidently explain the model.**

---

## 4) Splits that respect geography and time

Geospatial data is spatially autocorrelated.

Nearby observations often look similar and share context.

Better validation splits include:

- region-based split,
- scene-based split,
- time-based split,
- sensor-based split,
- holdout city or watershed,
- future-date test set.

Use the split that matches the deployment question.

Examples:

- If the model will run in new neighborhoods, hold out neighborhoods.
- If the model will run next season, hold out later dates.
- If the model will run on a new sensor, test on that sensor.

Core idea:

- **Validation should imitate the way the model will be used after training.**

---

## 5) Feature engineering beyond imagery

Not every geospatial ML problem is an image model.

Useful spatial features can come from:

- spatial joins,
- raster sampling,
- zonal statistics,
- distance to roads or water,
- neighborhood density,
- elevation and slope,
- land-cover proportions,
- weather or climate aggregates,
- temporal change metrics.

For tabular models, each prediction unit needs a stable feature row.

Examples:

- parcel risk score,
- road segment maintenance priority,
- watershed flood susceptibility,
- census-area heat vulnerability.

Check that every feature is available at inference time. A feature that exists only after the event may leak future information.

---

## 6) Training and evaluation workflow

A practical workflow usually starts simple.

Use:

- baseline model,
- clear metric,
- meaningful validation split,
- error analysis,
- documented assumptions.

For imagery models, evaluate:

- class performance,
- spatial generalization,
- edge cases,
- cloud or shadow sensitivity,
- performance by region or season.

For tabular spatial models, evaluate:

- feature leakage,
- spatial autocorrelation,
- calibration,
- residual maps,
- performance by subgroup or region.

Core idea:

- **A weaker model with honest validation is more useful than a stronger model with leaked validation.**

---

## 7) Inference and post-processing

Inference turns a trained model into predictions over the target geography.

Batch inference may need:

- tiling,
- windowed raster reads,
- distributed processing,
- model checkpoints,
- retry logic,
- output partitioning,
- progress tracking.

Post-processing may include:

- mosaicking chips,
- deduplicating detections,
- smoothing class maps,
- removing tiny artifacts,
- thresholding confidence,
- joining predictions back to features,
- validating geometry.

Keep the rules documented. Small cleanup choices can change counts, areas, and decisions.

---

## 8) Delivery formats

Predictions become useful when they reach the workflow that needs them.

Common delivery formats:

- GeoTIFF or Cloud Optimized GeoTIFF,
- GeoPackage,
- PostGIS table,
- vector tiles,
- raster tiles,
- GeoJSON API,
- dashboard summary,
- CSV with stable feature IDs.

Choose based on use:

- raster outputs for continuous surfaces or class maps,
- vector outputs for features and review workflows,
- database outputs for querying,
- tiles for display,
- APIs for product integration.

Core idea:

- **The final output should match how people inspect, query, update, and trust the prediction.**

---

## 9) Monitoring and maintenance

Applied ML does not end at the first prediction layer.

Monitor:

- data drift,
- sensor changes,
- seasonal changes,
- missing inputs,
- inference failures,
- confidence distribution,
- reviewer corrections,
- downstream user feedback.

Retraining may be needed when:

- new imagery sources arrive,
- geography changes,
- labels improve,
- model errors cluster in new regions,
- policy or class definitions change.

Document:

- model version,
- data version,
- known limitations,
- intended use,
- review process,
- update schedule.

---

## 10) Common anti-patterns

- Choosing a model before defining the output decision.
- Using random splits when deployment is spatial or temporal.
- Creating features that will not exist at inference time.
- Mixing label versions without tracking them.
- Publishing predictions without confidence or limitations.
- Treating post-processing rules as invisible cleanup.
- Ignoring reviewer feedback.
- Measuring only model metrics and not map usefulness.
- Losing CRS, transform, or feature IDs during inference.

---

## 11) Interview-ready Q&A

### Q1: How do you start a geospatial ML project?

Define the objective, prediction unit, target variable, output user, update frequency, validation strategy, and delivery format.

### Q2: Why are spatial splits important?

They test whether the model generalizes to new geography instead of memorizing nearby patterns.

### Q3: What is data leakage in geospatial ML?

It happens when training uses information that would not be available at prediction time, or when validation data is too spatially or temporally close to training data.

### Q4: What are examples of non-imagery geospatial features?

Distances, spatial joins, raster samples, zonal statistics, elevation, slope, land-cover proportions, density, and temporal aggregates.

### Q5: What makes a prediction map operational?

It has spatial metadata, confidence or uncertainty, provenance, validation, delivery format, and a process for review or update.

### Q6: Why does post-processing need documentation?

Because thresholds, cleanup, smoothing, and deduplication can materially change counts, areas, and decisions.

---

## 12) 30-minute mini exercise

Design an applied GeoAI pipeline for urban change detection:

1. Define the output user and decision.
2. Choose the prediction unit: pixel, building, parcel, or grid cell.
3. Source before-and-after imagery and reference labels.
4. Track imagery dates, CRS, resolution, and label versions.
5. Split validation by region or acquisition date.
6. Train a baseline model.
7. Run batch inference over the target area.
8. Post-process predictions into a map-ready layer.
9. Publish outputs as GeoTIFF, PostGIS, tiles, or an API.
10. Define monitoring, reviewer feedback, and retraining triggers.

Write the acceptance checks that would make the output safe to share.

---

## 13) One-page recap

- Start with the decision and output user.
- Choose the prediction unit before choosing the model.
- Track data, labels, chips, splits, and model versions.
- Use validation splits that match deployment.
- Geospatial ML can use imagery, vector features, rasters, and tabular models.
- Inference needs tiling, metadata, and post-processing rules.
- Delivery format should match the workflow.
- Monitoring and retraining keep predictions useful over time.
