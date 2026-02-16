# GEE Burn Severity (dNBR) + MODIS/FIRMS Validation — Horse River Fire (2016)

This project maps **burn severity** for the **2016 Horse River (Fort McMurray) wildfire** using **Google Earth Engine (GEE)**.  
It calculates **NBR (Normalized Burn Ratio)** before and after the fire using Landsat 8, then computes **dNBR = preNBR − postNBR** and classifies severity into **five classes (1–5)**.  
Results are cross-checked against independent datasets: **MODIS MCD64A1 burned area** and **FIRMS active fire detections**.

**Author:** Mohan Mahbaz  
**Study Area:** Fort McMurray / Wood Buffalo Region, Alberta (Horse River Fire AOI)  
**Year:** 2016

---

## Notebook / Code Link
- GEE script / notebook link: [Open in Google Earth Engine](https://code.earthengine.google.com/02147d9bd266efb1c29790bcadd6b2aa)

---

## Research Question
**What was the spatial pattern of burn severity for the 2016 Horse River wildfire (Fort McMurray), measured by dNBR, and how well does it agree with MODIS burned area and FIRMS active fire detections?**

---

## Data Used (all available in Google Earth Engine)

### 1) Landsat 8 Collection 2 Level 2 (Surface Reflectance)
- Used to compute **pre-fire NBR** and **post-fire NBR**
- Spatial resolution: **30 m**
- Dataset: `LANDSAT/LC08/C02/T1_L2`

### 2) MODIS Burned Area (MCD64A1)
- Used as an **independent burned/unburned reference**
- Burned pixels are identified where **BurnDate > 0**
- Dataset: `MODIS/061/MCD64A1`

### 3) FIRMS Active Fire (MODIS/VIIRS)
- Used as a **timing/consistency check** for active fire detections
- Dataset: `FIRMS`

---

## Key Concepts (simple definitions)

### NBR (Normalized Burn Ratio)
NBR highlights burned vegetation by comparing near-infrared and shortwave infrared reflectance:

**NBR = (NIR − SWIR2) / (NIR + SWIR2)**

For Landsat 8:
- **NIR = SR_B5**
- **SWIR2 = SR_B7**

### dNBR (Differenced NBR)
dNBR measures change caused by fire:

**dNBR = pre-fire NBR − post-fire NBR**

Large positive dNBR values typically indicate more severe burn effects.

---

## Method (What the code does, step-by-step)

### Step 0 — AOI preview (always works)
The script starts with a **preview AOI** (a point buffered by 200 km) so the map always shows something:

```js
var previewAOI = ee.Geometry.Point([-111.38, 56.73]).buffer(200000);
Map.centerObject(previewAOI, 7);
