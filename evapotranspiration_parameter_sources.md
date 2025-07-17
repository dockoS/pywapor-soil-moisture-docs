# Data Sources for PyWaPOR Evapotranspiration Parameters

This document explains where to obtain each parameter used in the PyWaPOR actual evapotranspiration calculation. For each parameter, we provide both the data source and the preprocessing method (if applicable).

## Table of Contents

1. [Radiation Parameters](#radiation-parameters)
2. [Surface Parameters](#surface-parameters)
3. [Meteorological Parameters](#meteorological-parameters)
4. [Resistance Parameters](#resistance-parameters)
5. [External Data Sources](#external-data-sources)
6. [Preprocessing Workflow](#preprocessing-workflow)

## Radiation Parameters

### Net Radiation Calculation

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `albedo` | **Sentinel-2 MSI bands** (10m resolution)<br>Calculate from surface reflectance | Quality filtering based on cloud masks<br>Apply atmospheric correction |
| `ra_hor_clear` | **ERA5-Land** (9km resolution)<br>Downward shortwave radiation at surface | Temporal interpolation to satellite overpass time<br>Spatial interpolation to match study area grid |
| `emiss_s` | **Derived from NDVI**<br>Surface emissivity calculation | Linear relationship with NDVI<br>Typical range: 0.94 (bare soil) to 0.99 (full vegetation) |
| `lst_k` | **Landsat 8/9 TIRS bands** (100m resampled)<br>Land surface temperature | Atmospheric correction<br>Emissivity correction<br>Convert from °C to K |
| `emiss_atm` | **Calculated from meteorological data**<br>Using atmospheric emissivity function | Requires air temperature, vapor pressure, and atmospheric pressure |
| `t_air_k` | **ERA5-Land** (9km resolution)<br>2m air temperature | Convert from °C to K<br>Temporal and spatial interpolation |

### Soil Heat Flux

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `rn_soil` | **Calculated**<br>Net radiation reaching soil surface | Based on net radiation and vegetation cover |
| `vc` | **Calculated from NDVI**<br>Vegetation cover fraction | Function of NDVI with thresholds |
| `g0` | **Default value** (typically 0.35 for bare soil)<br>Ratio of soil heat flux to net radiation | Can be calibrated based on soil type |

## Surface Parameters

### Vegetation Parameters

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `ndvi` | **Sentinel-2 MSI bands** (10m resolution)<br>Calculate as: (NIR - RED) / (NIR + RED) | Quality filtering based on cloud masks |
| `lai` | **Derived from NDVI**<br>Leaf Area Index | Empirical or semi-empirical relationship with NDVI |
| `h_veg` | **Derived from NDVI or LAI**<br>Vegetation height | Empirical relationship or lookup tables based on land cover |
| `f_c` | **Derived from NDVI**<br>Vegetation cover fraction | Similar to calculation for soil moisture |

### Soil Parameters

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `se_root` | **Previously calculated soil moisture**<br>Relative root zone soil moisture | Direct use from soil moisture calculation |
| `soil_type` | **SoilGrids or similar**<br>Soil texture classification | Resampling to match study area grid |
| `porosity` | **SoilGrids derived**<br>Soil porosity | Based on soil texture using pedotransfer functions |

## Meteorological Parameters

### Air Properties

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `ad_i` | **Calculated from meteorological data**<br>Using air density function | Requires air temperature, specific humidity, and pressure |
| `pressure` | **ERA5-Land** (9km resolution)<br>Surface pressure | Spatial interpolation and terrain adjustment |
| `vp_a` | **ERA5-Land derived** (9km resolution)<br>Actual vapor pressure | Calculate from dew point temperature or relative humidity |
| `vp_s` | **Calculated**<br>Saturation vapor pressure | Function of air temperature |
| `u_b` | **ERA5-Land** (9km resolution)<br>10m wind speed | Spatial interpolation and height adjustment |
| `precipitation` | **CHIRPS** (5km resolution)<br>Daily precipitation | Temporal aggregation and spatial interpolation |

## Resistance Parameters

### Aerodynamic Resistances

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `z0m` | **Derived from vegetation height**<br>Roughness length for momentum | Typically z0m = 0.123 × h_veg |
| `z0h` | **Derived from z0m**<br>Roughness length for heat | Typically z0h = 0.1 × z0m |
| `disp` | **Derived from vegetation height**<br>Displacement height | Typically disp = 0.67 × h_veg |
| `raa_e` | **Calculated**<br>Aerodynamic resistance for evaporation | Requires wind speed, stability parameters, and surface roughness |
| `raa_t` | **Calculated**<br>Aerodynamic resistance for transpiration | Similar to raa_e but for the canopy level |

### Surface Resistances

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `rss` | **Calculated from soil moisture**<br>Soil surface resistance | Exponential function of surface soil moisture |
| `rst` | **Calculated from multiple factors**<br>Stomatal resistance | Function of LAI, solar radiation, vapor pressure deficit, air temperature, and soil moisture |
| `rst_min` | **Lookup table by vegetation type**<br>Minimum stomatal resistance | Based on land cover or vegetation type |

## External Data Sources

### Satellite Data

| Data Type | Satellite Source | Resolution | Access |
|-----------|-----------------|------------|--------|
| Optical bands | **Sentinel-2 MSI** | 10m | [Copernicus Open Access Hub](https://scihub.copernicus.eu/) |
| Thermal bands | **Landsat 8/9 TIRS** | 100m resampled | [USGS Earth Explorer](https://earthexplorer.usgs.gov/) |
| Land Cover | **ESA WorldCover** | 10m | [ESA WorldCover](https://esa-worldcover.org/) |

### Meteorological Data

| Data Type | Source | Resolution | Access |
|-----------|--------|------------|--------|
| Air Temperature | **ERA5-Land** | 9km | [Copernicus Climate Data Store](https://cds.climate.copernicus.eu/) |
| Wind Speed | **ERA5-Land** | 9km | [Copernicus Climate Data Store](https://cds.climate.copernicus.eu/) |
| Solar Radiation | **ERA5-Land** | 9km | [Copernicus Climate Data Store](https://cds.climate.copernicus.eu/) |
| Precipitation | **CHIRPS** | 5km | [CHIRPS Website](https://www.chc.ucsb.edu/data/chirps) |

### Soil Data

| Data Type | Source | Resolution | Access |
|-----------|--------|------------|--------|
| Soil Properties | **SoilGrids** | 250m | [SoilGrids Website](https://soilgrids.org/) |
| Texture Classes | **FAO Harmonized World Soil Database** | 1km | [FAO HWSD](https://www.fao.org/soils-portal/data-hub/soil-maps-and-databases/harmonized-world-soil-database-v12/en/) |

## Preprocessing Workflow

To prepare all inputs for evapotranspiration calculation:

1. **Download raw data**:
   - Sentinel-2 L1C images (for NDVI, albedo)
   - Landsat 8/9 imagery (for LST)
   - ERA5-Land meteorological data
   - Precipitation data
   - Soil data

2. **Process optical data**:
   - Atmospheric correction of Sentinel-2 data
   - Calculate NDVI
   - Calculate albedo
   - Calculate LAI
   - Calculate vegetation cover

3. **Process thermal data**:
   - Calculate land surface temperature from Landsat thermal bands
   - Apply emissivity corrections

4. **Process meteorological data**:
   - Interpolate ERA5-Land to study area grid
   - Calculate derived parameters (air density, vapor pressures, etc.)

5. **Prepare resistance parameters**:
   - Calculate aerodynamic resistances
   - Calculate surface resistances

6. **Radiation components**:
   - Calculate net radiation components
   - Partition between soil and vegetation

7. **Calculate ET components**:
   - Soil evaporation
   - Transpiration
   - Interception evaporation

8. **Calculate total ET**:
   - Combine ET components
   - Convert to appropriate units (mm/day)

9. **Apply downscaling** (if higher resolution is needed):
   - Use thermal sharpening or machine learning approaches
