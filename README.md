# Project 4 — GEE Burn Severity (dNBR) + MODIS/FIRMS Validation  
**Horse River Wildfire (Fort McMurray), 2016**

This project maps wildfire burn severity for the **2016 Horse River Fire** (Fort McMurray / Wood Buffalo, Alberta) using **dNBR (difference Normalized Burn Ratio)** computed from **Landsat 8 Surface Reflectance** in **Google Earth Engine (GEE)**.  
Results are validated using two independent sources:

- **MODIS MCD64A1 Burned Area** (burned / unburned)
- **NASA FIRMS active fire detections** (high-confidence detections during the fire window)

**Author:** Mohan Mahbaz  
**Tools:** Google Earth Engine (Code Editor), ArcGIS Pro (map layout + cartography)

---

## Research Question
**What is the spatial pattern of burn severity (dNBR) for the Horse River Fire (2016), and how well does it match MODIS burned-area mapping and FIRMS active-fire detections?**

---

## Study Area
Fort McMurray / Regional Municipality of Wood Buffalo, Alberta.

- You can **draw the AOI polygon** directly in the GEE Code Editor (left toolbar).
- If no polygon is drawn, the script uses a **fallback preview AOI** so the map always runs.

---

## Notebook / Code Link

- **GEE Script / Notebook:** https://code.earthengine.google.com/02147d9bd266efb1c29790bcadd6b2aa


---

## Data Used (Google Earth Engine)
### Landsat (Burn Severity Mapping)
- **Landsat 8 Collection 2, Level-2 Surface Reflectance**  
  `LANDSAT/LC08/C02/T1_L2`  
  Used to compute pre-fire NBR, post-fire NBR, and dNBR at **30 m** resolution.

### MODIS (Burned/Unburned Validation)
- **MODIS MCD64A1 Burned Area**  
  `MODIS/061/MCD64A1`  
  Used as an independent burned/unburned reference at **500 m** resolution.

### FIRMS (Active Fire Cross-Check)
- **FIRMS Active Fire**  
  `FIRMS`  
  Used to show high-confidence fire detections during the main activity window.

---

## Outputs (Final Maps)
All final map layouts are stored in the `images/` folder:

- Pre-Fire NBR — `images/Pre-Fire NBR.pdf`
- Post-Fire NBR — `images/Post-Fire NBR.pdf`
- dNBR — `images/DNBR.pdf`
- Burn Severity Classes (1–5) — `images/Burn severity classes.pdf`
- MODIS Burned Area (MCD64A1, 2016) — `images/MODIS.pdf`
- dNBR Burned Mask (≥ Low Severity) — `images/DNBR Burned mask.pdf`

---

# Method (What the code does, step-by-step)

## Step 0 — AOI Preview (Always Works)
This creates a **preview AOI** using a point near Fort McMurray buffered by **200 km**, then zooms the map to it.  
This guarantees the script shows something even if you haven’t drawn an AOI polygon yet.

```js
// -----------------------------
// AOI + MODIS preview (always works)
// -----------------------------
var previewAOI = ee.Geometry.Point([-111.38, 56.73]).buffer(200000); // 200 km preview
Map.centerObject(previewAOI, 7);

Step 1 — MODIS Burned-Area Preview (Helps You Draw the AOI)

This pulls MODIS MCD64A1 burned area for the 2016 season window and displays burned pixels in blue.
It helps you draw the AOI polygon around the actual burned footprint.

// MODIS burned-area preview (blue) to help you draw the AOI
var mcdPreview = ee.ImageCollection('MODIS/061/MCD64A1')
  .filterBounds(previewAOI)
  .filterDate('2016-05-01', '2016-09-30')
  .select('BurnDate')
  .max();

Map.addLayer(mcdPreview.gt(0).selfMask(), {palette:['0000ff']},
  'MCD64A1 burned (2016) preview'
);


Interpretation:

BurnDate > 0 means MODIS recorded a burn during the time window.

.selfMask() hides zeros so only burned pixels display.

Step 2 — Use Your Drawn AOI (Or Fallback to Preview AOI)

If you draw a polygon in the GEE Code Editor, it is typically named geometry, geometry2, or geometry3.
This logic automatically picks the drawn polygon if it exists; otherwise it uses previewAOI.

// If you drew a polygon, GEE names it geometry / geometry2 / geometry3.
// This picks it automatically; otherwise it falls back to previewAOI.
var aoi =
  (typeof geometry  !== 'undefined') ? geometry  :
  (typeof geometry2 !== 'undefined') ? geometry2 :
  (typeof geometry3 !== 'undefined') ? geometry3 :
  previewAOI;

Map.addLayer(aoi, {color: 'yellow'}, 'AOI');

Step 3 — Preview MODIS Again (Now Using the Final AOI)

Once the AOI is set, the script repeats the MODIS burned-area preview but clipped to the AOI.

// -----------------------------
// Preview MODIS MCD64A1 burned area (to help redraw AOI)
// -----------------------------
var mcdPreview = ee.ImageCollection('MODIS/061/MCD64A1')
  .filterBounds(aoi)
  .filterDate('2016-05-01', '2016-09-30')
  .select('BurnDate')
  .max();

Map.addLayer(mcdPreview.gt(0).selfMask(), {palette:['0000ff']},
  'MCD64A1 burned (2016) preview'
);

Step 4 — Load Landsat 8 SR and Filter Scenes

This loads Landsat 8 L2 Surface Reflectance, limits to the AOI, and keeps scenes with < 30% cloud cover.

// -----------------------------
// Landsat 8 SR (pre & post fire windows for 2016)
// -----------------------------
var l8 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
  .filterBounds(aoi)
  .filter(ee.Filter.lt('CLOUD_COVER', 30));

Step 5 — Cloud Masking (QA_PIXEL)

This function masks out clouds and cloud contamination using the QA_PIXEL bit flags.

function maskL8L2(img) {
  var qa = img.select('QA_PIXEL');
  var mask = qa.bitwiseAnd(1<<1).eq(0)
    .and(qa.bitwiseAnd(1<<2).eq(0))
    .and(qa.bitwiseAnd(1<<3).eq(0))
    .and(qa.bitwiseAnd(1<<4).eq(0))
    .and(qa.bitwiseAnd(1<<5).eq(0));
  return img.updateMask(mask);
}


Why this matters:
Clouds and cloud shadows can create false burn signals in NBR and dNBR. Masking improves reliability.

Step 6 — Prepare Landsat Reflectance and Compute NBR

Landsat L2 surface reflectance bands require scaling to convert stored values into real reflectance.
Then NBR is computed using NIR and SWIR2:

NIR: SR_B5

SWIR2: SR_B7

NBR: (NIR − SWIR2) / (NIR + SWIR2)

function prepL8(img){
  img = maskL8L2(img);

  // Convert DN-like stored values into reflectance:
  var sr = img.select(['SR_B.*']).multiply(0.0000275).add(-0.2);

  // NBR = (NIR - SWIR2) / (NIR + SWIR2)
  var nbr = sr.normalizedDifference(['SR_B5','SR_B7']).rename('NBR');

  return nbr.copyProperties(img, ['system:time_start']);
}

Step 7 — Build Pre-Fire and Post-Fire Median Composites

To reduce cloud noise and scene-level artifacts, the script uses a median composite for each time window:

Pre-fire window: 2016-04-01 to 2016-05-02

Post-fire window: 2016-06-01 to 2016-09-30

var pre  = l8.filterDate('2016-04-01', '2016-05-02').map(prepL8).median().clip(aoi);
var post = l8.filterDate('2016-06-01', '2016-09-30').map(prepL8).median().clip(aoi);

Step 8 — Compute dNBR

dNBR is computed as:

dNBR = preNBR − postNBR

High positive values typically indicate greater burn-related change.

var dNBR = pre.subtract(post).rename('dNBR');

Map.addLayer(dNBR, {min: -0.2, max: 1.0, palette: ['white','yellow','orange','red','maroon']},
  'dNBR (Landsat 8)'
);

print('Pre NBR', pre);
print('Post NBR', post);
print('dNBR', dNBR);

Step 9 — Classify Burn Severity into 5 Classes

This turns continuous dNBR into severity classes (1–5) using standard thresholds:

1: dNBR < 0.10

2: 0.10–0.27

3: 0.27–0.44

4: 0.44–0.66

5: > 0.66

// Burn severity classes
var severity = dNBR.expression(
  "(b('dNBR') < 0.1) ? 1" +
  ": (b('dNBR') < 0.27) ? 2" +
  ": (b('dNBR') < 0.44) ? 3" +
  ": (b('dNBR') < 0.66) ? 4" +
  ": 5"
).rename('severity');

Map.addLayer(severity, {min: 1, max: 5, palette: ['ffffff','7fffd4','ffff00','ff9900','ff0000']},
  'Burn severity (classes)'
);

Step 10 — Area Statistics by Severity (Hectares)

This computes area per class in hectares:

ee.Image.pixelArea() gives square meters per pixel

Divide by 10,000 to convert to hectares

Group results by severity class

// Area stats (hectares)
var areaImg = ee.Image.pixelArea().divide(10000).addBands(severity);

var areas = areaImg.reduceRegion({
  reducer: ee.Reducer.sum().group({groupField: 1, groupName: 'severity'}),
  geometry: aoi,
  scale: 30,
  maxPixels: 1e13
});
print('Area (ha) by severity class', areas);

// Convert grouped dictionary to a FeatureCollection for charting
var areaList = ee.List(areas.get('groups'));
var areaFc = ee.FeatureCollection(areaList.map(function(d){
  d = ee.Dictionary(d);
  return ee.Feature(null, {severity: d.get('severity'), ha: d.get('sum')});
}));

print(ui.Chart.feature.byFeature(areaFc, 'severity', 'ha').setChartType('ColumnChart'));

Step 11 — Validation vs MODIS Burned/Unburned (MCD64A1)

This builds a MODIS burned/unburned raster:

modisBurn = 1 if BurnDate > 0

modisBurn = 0 otherwise

Then it builds a GEE burned mask:

geeBurn = 1 if severity ≥ 2 (≥ low severity)

geeBurn = 0 otherwise

// Validation vs MODIS burned/unburned
var mcd = ee.ImageCollection('MODIS/061/MCD64A1')
  .filterBounds(aoi)
  .filterDate('2016-05-01', '2016-09-30')
  .select('BurnDate')
  .max()
  .clip(aoi);

var modisBurn = mcd.gt(0).unmask(0).rename('modisBurn');
var geeBurn = severity.gte(2).unmask(0).rename('geeBurn');  // >= low severity

Map.addLayer(modisBurn.selfMask(), {palette:['0000ff']}, 'MCD64A1 burned (2016)');
Map.addLayer(geeBurn.selfMask(),   {palette:['00ff00']}, 'dNBR burned (>=low)');

Confusion matrix + accuracy

This samples points stratified by MODIS burned/unburned and computes accuracy.

var validation = ee.Image.cat(modisBurn, geeBurn).stratifiedSample({
  numPoints: 1000,
  region: aoi,
  classBand: 'modisBurn',
  classValues: [0, 1],
  classPoints: [1000, 1000],
  scale: 500,
  seed: 42,
  geometries: false,
  tileScale: 4
});

var cm = validation.errorMatrix('modisBurn', 'geeBurn');
print('Confusion matrix (MODIS vs GEE burned)', cm);
print('Overall accuracy', cm.accuracy());
print('Kappa', cm.kappa());
print('Producers accuracy (by class)', cm.producersAccuracy());
print('Consumers accuracy (by class)', cm.consumersAccuracy());
print('F1 score', cm.fscore());

Step 12 — Active Fire Cross-Check (FIRMS)

This pulls FIRMS detections for the main activity window and maps only confidence ≥ 80.

// Active fire cross-check (FIRMS)
var fires = ee.ImageCollection('FIRMS')
  .filterBounds(aoi)
  .filterDate('2016-05-01', '2016-06-30');   // fire activity window

// Take max confidence over the window (0–100)
var fireConf = fires.select('confidence').max().clip(aoi);

// Show only higher-confidence detections
var fireMask = fireConf.gte(80).selfMask();
Map.addLayer(fireMask, {palette:['ff00ff']}, 'FIRMS active fires (conf >= 80)');

Optional overlap check

This estimates how often FIRMS hotspots fall inside the burned mask.

var burned = geeBurn; // 0/1 burned mask from severity
var overlap = burned.updateMask(fireMask).reduceRegion({
  reducer: ee.Reducer.mean(),
  geometry: aoi,
  scale: 500,
  maxPixels: 1e13
});
print('Fraction of FIRMS-fire pixels classified burned (>=low)', overlap);

Step 13 — Exports (GeoTIFF + CSV)

These exports send raster outputs and area stats to Google Drive.

// EXPORTS (GeoTIFF + CSV)
Export.image.toDrive({
  image: pre.rename('preNBR'),
  description: 'FortMcMurray_preNBR_2016_L8',
  region: aoi,
  scale: 30,
  maxPixels: 1e13
});

Export.image.toDrive({
  image: post.rename('postNBR'),
  description: 'FortMcMurray_postNBR_2016_L8',
  region: aoi,
  scale: 30,
  maxPixels: 1e13
});

Export.image.toDrive({
  image: dNBR,
  description: 'FortMcMurray_dNBR_2016_L8',
  region: aoi,
  scale: 30,
  maxPixels: 1e13
});

Export.image.toDrive({
  image: severity.toInt8(),
  description: 'FortMcMurray_burnSeverityClasses_2016_L8',
  region: aoi,
  scale: 30,
  maxPixels: 1e13
});

Export.table.toDrive({
  collection: areaFc,
  description: 'FortMcMurray_area_ha_by_severity',
  fileFormat: 'CSV'
});

Exports for ArcGIS Pro validation maps

These are simple 0/1 rasters for comparing burned area footprints in ArcGIS Pro.

// EXPORTS for Validation Map (ArcGIS Pro)
// 1) MODIS MCD64A1 burned/unburned (0/1)
// 2) dNBR burned mask (>= Low severity) (0/1)

Export.image.toDrive({
  image: modisBurn.toByte(),
  description: 'FortMcMurray_MODIS_MCD64A1_burned_2016_01',
  region: aoi,
  scale: 500,
  maxPixels: 1e13
});

Export.image.toDrive({
  image: geeBurn.toByte(),
  description: 'FortMcMurray_dNBR_burnMask_geLow_2016_01',
  region: aoi,
  scale: 30,
  maxPixels: 1e13
});

Key Takeaway

The dNBR burn severity map identifies where the strongest spectral changes occurred between pre- and post-fire conditions.
Validation with MODIS burned area and FIRMS active fires provides an independent check that the mapped footprint and timing are consistent with other remote-sensing products.

Limitations

Cloud/smoke sensitivity: Even with masking, some artifacts can remain.

Seasonal effects: Vegetation phenology can affect NBR if time windows differ in greenness.

Resolution mismatch: MODIS burned area is 500 m, Landsat is 30 m; edges will not match perfectly.

Thresholds: Severity thresholds are standard and may require local calibration for the best results.

FIRMS meaning: FIRMS shows hotspots (active fire), not final burn severity.

## Repository Structure

- `README.md` — project documentation (this file)
- `images/` — exported final map layouts (PDF):
  - `images/Burn severity classes.pdf` — dNBR severity classes (1–5)
  - `images/DNBR Burned mask.pdf` — binary burned/unburned mask (≥ low severity)
  - `images/DNBR.pdf` — continuous dNBR surface
  - `images/MODIS.pdf` — MODIS MCD64A1 burned-area product (reference/validation)
  - `images/Pre-Fire NBR.pdf` — pre-fire NBR composite
  - `images/Post-Fire NBR.pdf` — post-fire NBR composite
