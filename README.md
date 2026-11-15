# ECOSTRESS Collection 2 Data Product Granule Encapsulation Classes (`ECOv002-granules`)

The `ECOv002-granules` Python package provides object-oriented encapsulation classes, readers, writers, and visualization utilities for NASA ECOSTRESS Collection 2 tiled data products. Built on top of the `rasters` library, these classes abstract away directory and `.zip` archive structures, Cloud Optimized GeoTIFF (COG) layer URIs, metadata parsing, spatial reprojection, unit conversions, and browse image generation.

**Author:** [Gregory H. Halverson](https://github.com/gregory-halverson-jpl) (<gregory.h.halverson@jpl.nasa.gov>), NASA Jet Propulsion Laboratory  
**License:** Apache License 2.0

---

## Installation

Requires Python 3.10 or higher. Install via `pip`:

```bash
pip install ECOv002-granules
```

To install with development and testing dependencies (`pytest`, `build`, `jupyter`, etc.):

```bash
pip install "ECOv002-granules[dev]"
```

---

## Architecture & Class Hierarchy

All tiled granule classes inherit from `ECOSTRESSTiledGranule` (which extends the abstract base class `ECOSTRESSGranule`):

* **`ECOSTRESSGranule`**: Defines the core interface for granule identification (orbit, scene, product, timestamp, build, process count), spatial boundaries, JSON metadata serialization, and PNG browse image rendering.
* **`ECOSTRESSTiledGranule`**: Implements tiled product storage handling—supporting both extracted directories and `.zip` archives transparently—along with Zstandard-compressed COG layer reading/writing, solar time conversion, and multi-granule mosaicking.
* **Product-Specific Subclasses**: Provide strongly-typed `@property` accessors, domain-specific colormaps, automatic cloud/water masking, and unit conversions for each ECOSTRESS Collection 2 product level.

| Class | Product Code | Short Name | Primary Variable | Description |
| :--- | :--- | :--- | :--- | :--- |
| **`L1CTRAD`** | `L1CT_RAD` | `ECO_L1CT_RAD` | `false_color` | Tiled Top-of-Atmosphere Calibrated Radiance (70 m) |
| **`L2TLSTE`** | `L2T_LSTE` | `ECO_L2T_LSTE` | `LST` | Tiled Land Surface Temperature and Emissivity (70 m) |
| **`L2TSTARS`** | `L2T_STARS` | `ECO_L2T_STARS` | `NDVI` | Tiled Surface Reflectance / NDVI and Albedo (70 m) |
| **`L3TJET`** | `L3T_JET` | `ECO_L3T_JET` | `ETdaily` | Tiled Evapotranspiration Ensemble Instantaneous & Daytime (70 m) |
| **`L3TMET`** | `L3T_MET` | `ECO_L3T_MET` | `Ta` | Tiled Downscaled Meteorology (70 m) |
| **`L3TSEB`** | `L3T_SEB` | `ECO_L3T_SEB` | `Rn` | Tiled Surface Energy Balance (70 m) |
| **`L3TSM`** | `L3T_SM` | `ECO_L3T_SM` | `SM` | Tiled Downscaled Soil Moisture (70 m) |
| **`L4TESI`** | `L4T_ESI` | `ECO_L4T_ESI` | `ESI` | Tiled Evaporative Stress Index (70 m) |
| **`L4TWUE`** | `L4T_WUE` | `ECO_L4T_WUE` | `WUE` | Tiled Water Use Efficiency (70 m) |

---

## Opening Granules & Common Operations

### Automatic Granule Detection (`open_granule`)

Use `open_granule()` to automatically inspect a filename and instantiate the appropriate product encapsulation class (`L2TLSTE`, `L2TSTARS`, `L3TJET`, `L3TMET`, `L3TSEB`, `L3TSM`, `L4TESI`, or `L4TWUE`). It works seamlessly on both `.zip` archives and unzipped granule directories.

```python
from ECOv002_granules import open_granule

granule = open_granule("ECOv002_L3T_JET_31245_007_11SLT_20240715T193022_0712_01.zip")
print(type(granule))  # <class 'ECOv002_granules.L3TJET.L3TJET'>
```

*(Note: `L1CTRAD` can be imported directly via `from ECOv002_granules.L1CTRAD import L1CTRAD`.)*

### Batch Directory Scanning (`scan_directory`)

Any tiled class can scan a directory and return a list of valid granule instances matching that class:

```python
from ECOv002_granules import L3TJET

jet_granules = L3TJET.scan_directory("/path/to/ecostress/data")
```

### Spatial & Temporal Properties

Every `ECOSTRESSTiledGranule` exposes parsed filename attributes, geometry objects, and solar time calculations:

```python
# Identifier attributes
print(granule.granule_name)    # "ECOv002_L3T_JET_31245_007_11SLT_20240715T193022_0712_01"
print(granule.product)         # "L3T_JET"
print(granule.orbit)           # 31245
print(granule.scene)           # 7
print(granule.tile)            # "11SLT"
print(granule.build)           # "0712"
print(granule.process_count)   # 1

# Temporal properties
print(granule.time_UTC)        # datetime(2024, 7, 15, 19, 30, 22)
print(granule.date_UTC)        # date(2024, 7, 15)
print(granule.time_solar)      # Local solar datetime adjusted by centroid longitude
print(granule.hour_of_day)     # Local solar hour as float (e.g., 11.65)

# Spatial properties (powered by `rasters` and `shapely`)
print(granule.geometry)        # RasterGrid object (70m UTM grid)
print(granule.bbox)            # Lat/lon BBox
print(granule.centroid)        # Lat/lon Point
print(granule.boundary)        # Shapely Polygon
print(granule.boundary_WKT)    # WKT string of the footprint
```

### Reading & Reprojecting Variables

You can inspect available layers in a granule and read any layer as a `rasters.Raster` object (with `np.nan` automatically assigned as `nodata` for floating-point layers). Passing a target `RasterGeometry` reprojects/resamples the layer on the fly:

```python
# List all available GeoTIFF variables inside the granule
print(granule.variables)

# Get the GDAL/rasterio URI (works directly inside .zip archives via zip://...)
print(granule.layer_URI("ETdaily"))

# Read a variable as a Raster
et_daily = granule.variable("ETdaily")

# Read and reproject to a custom target RasterGeometry
et_reprojected = granule.variable("ETdaily", geometry=target_geometry)
```

### Mosaicking Multiple Granules

Combine multiple tiles or scenes onto a single target `RasterGeometry` using the class method `mosaic()`:

```python
from ECOv002_granules import L3TJET

mosaicked_et = L3TJET.mosaic(
    variable="ETdaily",
    granules=jet_granules,
    geometry=regional_geometry
)
```

### Metadata & Browse Images

Access standard and product metadata dictionaries, export metadata to JSON, or generate quicklook PNG browse images:

```python
# Metadata dictionaries
std_meta = granule.standard_metadata
prod_meta = granule.product_metadata
full_meta = granule.metadata_dict

# Serialize or write JSON metadata
json_str = granule.get_metadata_JSON(indent=2)
granule.write_metadata_JSON("metadata.json")

# Generate a PIL Image or save a PNG browse image
pil_img = granule.browse_image
granule.write_browse_image("preview.png")
```

---

## Detailed Guide to Product Encapsulation Classes

### 1. `L1CTRAD` — Top-of-Atmosphere Calibrated Radiance (`L1CT_RAD`)

Encapsulates 5-band TIR calibrated radiance tiles and quality/mask layers.

* **Import:** `from ECOv002_granules.L1CTRAD import L1CTRAD`
* **Primary Variable:** `false_color`
* **Available Variables:** `radiance_1` through `radiance_5`, `data_quality_1` through `data_quality_5`, `water`, `cloud`
* **Methods & Properties:**
  * `granule.radiance(band: int, apply_cloud: bool = False) -> Raster`: Retrieves calibrated radiance for bands `1`–`5`.
  * `granule.false_color -> MultiRaster`: Stacks bands `[5, 4, 2]` into a 3-band false-color composite.
  * `granule.water -> Raster`: Water mask layer.
  * `granule.cloud -> Raster`: Cloud mask layer.

```python
from ECOv002_granules.L1CTRAD import L1CTRAD

l1 = L1CTRAD("ECOv002_L1CT_RAD_31245_007_11SLT_20240715T193022_0712_01.zip")
band5 = l1.radiance(5)
rgb_composite = l1.false_color
```

---

### 2. `L2TLSTE` — Land Surface Temperature & Emissivity (`L2T_LSTE`)

Encapsulates 70 m Land Surface Temperature, broadband emissivity, elevation, view geometry, and quality flags.

* **Import:** `from ECOv002_granules import L2TLSTE`
* **Primary Variable:** `LST`
* **Available Variables:** `LST`, `LST_err`, `EmisWB`, `water`, `cloud`, `height`, `view_zenith`, `QC`
* **Properties & Unit Conversions:**
  * `granule.LST` / `granule.ST_K`: Land surface temperature in **Kelvin** (automatically masks out cloudy pixels as `np.nan`).
  * `granule.ST_C`: Surface temperature converted to **Celsius** (`ST_K - 273.15`).
  * `granule.WST_C`: Water surface temperature in **Celsius** (masked to water pixels only).
  * `granule.LST_err`: Surface temperature uncertainty.
  * `granule.EmisWB` / `granule.emissivity`: Wideband emissivity.
  * `granule.height` / `granule.elevation_m`: Surface elevation in **meters**.
  * `granule.elevation_km`: Surface elevation in **kilometers**.
  * `granule.view_zenith`: Sensor view zenith angle.
  * `granule.water` / `granule.cloud`: Boolean masks (`dtype=bool`).
  * `granule.QC`: Quality control bitmask (`dtype=uint16`).

```python
from ECOv002_granules import L2TLSTE

l2 = L2TLSTE("ECOv002_L2T_LSTE_31245_007_11SLT_20240715T193022_0712_01.zip")
temp_celsius = l2.ST_C
water_temp_c = l2.WST_C
emissivity = l2.emissivity
elev_km = l2.elevation_km
```

---

### 3. `L2TSTARS` — Surface Reflectance & Vegetation Indices (`L2T_STARS`)

Encapsulates gap-filled/fused NDVI and broadband albedo along with their uncertainty layers. Note that `L2T_STARS` granules are tile-based composites without individual orbit/scene numbers (`granule.orbit` and `granule.scene` return `None`).

* **Import:** `from ECOv002_granules import L2TSTARS`
* **Primary Variable:** `NDVI` (uses custom `NDVI_COLORMAP`)
* **Properties:**
  * `granule.NDVI`: Normalized Difference Vegetation Index (with `NDVI_COLORMAP` attached).
  * `granule.NDVI_UQ`: NDVI uncertainty (`NDVI-UQ` layer).
  * `granule.albedo`: Surface broadband albedo (with `ALBEDO_COLORMAP` attached).
  * `granule.albedo_UQ`: Albedo uncertainty (`albedo-UQ` layer).

```python
from ECOv002_granules import L2TSTARS

stars = L2TSTARS("ECOv002_L2T_STARS_11SLT_20240715T000000_0712_01.zip")
ndvi = stars.NDVI
albedo = stars.albedo
```

---

### 4. `L3TJET` — Evapotranspiration Ensemble (`L3T_JET`)

Encapsulates multi-model instantaneous latent heat flux, daily evapotranspiration, and canopy/soil/interception partitioning. All ET layers default to `ET_COLORMAP`.

* **Import:** `from ECOv002_granules import L3TJET`
* **Primary Variable:** `ETdaily`
* **Properties & Unit Conversions:**
  * `granule.ETdaily`: Daily evapotranspiration.
  * `granule.ETinst` / `granule.LE`: Instantaneous latent heat flux ($\text{W m}^{-2}$).
  * `granule.ETinst_kg`: Instantaneous evapotranspiration mass flux ($\text{kg m}^{-2}\text{ s}^{-1}$, computed as `ETinst / 2450000.0`).
  * `granule.PTJPLSMinst`: Instantaneous ET from PT-JPL-SM.
  * `granule.PTJPLSMcanopy`: Canopy transpiration component (PT-JPL-SM).
  * `granule.PTJPLSMsoil`: Soil evaporation component (PT-JPL-SM).
  * `granule.PTJPLSMinterception`: Interception evaporation component (PT-JPL-SM).
  * `granule.STICinst`: Instantaneous ET from STIC.
  * `granule.STICcanopy`: Canopy transpiration component (STIC).
  * `granule.BESSinst`: Instantaneous ET from BESS.
  * `granule.MOD16inst`: Instantaneous ET from MOD16 / PM-JPL.

```python
from ECOv002_granules import L3TJET

jet = L3TJET("ECOv002_L3T_JET_31245_007_11SLT_20240715T193022_0712_01.zip")
daily_et = jet.ETdaily
le_watts = jet.LE
et_mass_flux = jet.ETinst_kg
transpiration = jet.PTJPLSMcanopy
```

---

### 5. `L3TMET` — Downscaled Meteorology (`L3T_MET`)

Encapsulates downscaled near-surface air temperature and relative humidity.

* **Import:** `from ECOv002_granules import L3TMET`
* **Primary Variable:** `Ta`
* **Layer Access:** Access meteorological variables via `.variable()`, such as `Ta` / `Ta_C` (air temperature), `RH` (relative humidity), `water`, and `cloud`.

```python
from ECOv002_granules import L3TMET

met = L3TMET("ECOv002_L3T_MET_31245_007_11SLT_20240715T193022_0712_01.zip")
air_temp = met.variable("Ta_C")
rel_humidity = met.variable("RH")
```

---

### 6. `L3TSEB` — Surface Energy Balance (`L3T_SEB`)

Encapsulates surface radiative fluxes including net radiation and incoming solar radiation.

* **Import:** `from ECOv002_granules import L3TSEB`
* **Primary Variable:** `Rn`
* **Properties & Layers:**
  * `granule.Rn`: Instantaneous net radiation ($\text{W m}^{-2}$).
  * Additional layers via `.variable()`: `Rg` (global solar radiation), `water`, `cloud`.

```python
from ECOv002_granules import L3TSEB

seb = L3TSEB("ECOv002_L3T_SEB_31245_007_11SLT_20240715T193022_0712_01.zip")
net_rad = seb.Rn
solar_rad = seb.variable("Rg")
```

---

### 7. `L3TSM` — Downscaled Soil Moisture (`L3T_SM`)

Encapsulates 70 m downscaled volumetric surface soil moisture.

* **Import:** `from ECOv002_granules import L3TSM`
* **Primary Variable:** `SM`
* **Properties & Layers:**
  * `granule.SM`: Volumetric soil moisture.
  * Additional layers via `.variable()`: `water`, `cloud`.

```python
from ECOv002_granules import L3TSM

sm = L3TSM("ECOv002_L3T_SM_31245_007_11SLT_20240715T193022_0712_01.zip")
soil_moisture = sm.SM
```

---

### 8. `L4TESI` — Evaporative Stress Index (`L4T_ESI`)

Encapsulates the ratio of actual to potential evapotranspiration (ESI) and potential evapotranspiration (PET).

* **Import:** `from ECOv002_granules import L4TESI`
* **Primary Variable:** `ESI`
* **Layer Access:** Access `ESI`, `PET`, `water`, and `cloud` via `.variable()`.

```python
from ECOv002_granules import L4TESI

esi_granule = L4TESI("ECOv002_L4T_ESI_31245_007_11SLT_20240715T193022_0712_01.zip")
esi = esi_granule.variable("ESI")
pet = esi_granule.variable("PET")
```

---

### 9. `L4TWUE` — Water Use Efficiency (`L4T_WUE`)

Encapsulates ecosystem Water Use Efficiency (WUE) and Gross Primary Productivity (GPP) with built-in carbon flux unit conversions.

* **Import:** `from ECOv002_granules import L4TWUE`
* **Primary Variable:** `WUE` (uses `GPP_COLORMAP`)
* **Properties & Unit Conversions:**
  * `granule.WUE`: Water Use Efficiency.
  * `granule.GPP` / `granule.GPP_umol`: Gross Primary Productivity in $\mu\text{mol C m}^{-2}\text{ s}^{-1}$.
  * `granule.GPP_mol`: Gross Primary Productivity converted to $\text{mol C m}^{-2}\text{ s}^{-1}$ (`GPP_umol / 1e6`).
  * `granule.GPP_g`: Gross Primary Productivity converted to $\text{g C m}^{-2}\text{ s}^{-1}$ (`GPP_mol * 12.011`).

```python
from ECOv002_granules import L4TWUE

wue_granule = L4TWUE("ECOv002_L4T_WUE_31245_007_11SLT_20240715T193022_0712_01.zip")
wue = wue_granule.WUE
gpp_grams = wue_granule.GPP_g
```

---

## Creating & Writing New Collection 2 Products

The package also provides high-level writer functions and low-level granule builders for generating standard ECOSTRESS Collection 2 `.zip` archives, `.zip.xml` CAS metadata sidecars, and `.png` browse images.

### High-Level Product Writers

Six dedicated functions handle water masking, `float32`/`uint8` casting, QA percentage calculation, COG layer creation, metadata updates, zip archiving, XML sidecar generation, and temporary directory cleanup:

* `write_L3T_JET(...)`
* `write_L3T_MET(...)`
* `write_L3T_SEB(...)`
* `write_L3T_SM(...)`
* `write_L4T_ESI(...)`
* `write_L4T_WUE(...)`

```python
from datetime import datetime
from ECOv002_granules import write_L3T_SM

write_L3T_SM(
    L3T_SM_zip_filename="output/ECOv002_L3T_SM_31245_007_11SLT_20240715T193022_0712_01.zip",
    L3T_SM_browse_filename="output/ECOv002_L3T_SM_31245_007_11SLT_20240715T193022_0712_01.png",
    L3T_SM_directory="tmp/ECOv002_L3T_SM_31245_007_11SLT_20240715T193022_0712_01",
    orbit=31245,
    scene=7,
    tile="11SLT",
    time_UTC=datetime(2024, 7, 15, 19, 30, 22),
    build="0712",
    product_counter=1,
    SM=sm_raster,
    water_mask=water_raster,
    cloud_mask=cloud_raster,
    metadata=metadata_dict
)
```

### Custom Granule Assembly

You can also instantiate any tiled class directly by specifying its orbit, scene, tile, timestamp, build, and process count, and add custom COG layers manually:

```python
from datetime import datetime
from ECOv002_granules import L3TSM, SM_COLORMAP

granule = L3TSM(
    orbit=31245,
    scene=7,
    tile="11SLT",
    time_UTC=datetime(2024, 7, 15, 19, 30, 22),
    build="0712",
    process_count=1,
    containing_directory="./staging"
)

granule.add_layer("SM", sm_raster, cmap=SM_COLORMAP)
granule.write_metadata(metadata_dict)
granule.write_zip("ECOv002_L3T_SM_31245_007_11SLT_20240715T193022_0712_01.zip")
granule.write_browse_image("ECOv002_L3T_SM_31245_007_11SLT_20240715T193022_0712_01.png")
```

---

## Colormaps

`ECOv002_granules` exports predefined `matplotlib` `LinearSegmentedColormap` objects tailored to each geophysical variable:

* `ET_COLORMAP` (Evapotranspiration / ESI / PET)
* `SM_COLORMAP` & `RH_COLORMAP` (Soil Moisture & Relative Humidity)
* `NDVI_COLORMAP` / `NDVI_COLORMAP_ABSOLUTE` (Vegetation Index)
* `ALBEDO_COLORMAP` (Surface Albedo)
* `GPP_COLORMAP` (Gross Primary Productivity & Water Use Efficiency)
* `WATER_COLORMAP` & `CLOUD_COLORMAP` / `CLOUD_CMAP` (Binary Masks)