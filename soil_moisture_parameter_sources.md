# Data Sources for PyWaPOR Soil Moisture Parameters

This document explains where to obtain each parameter used in the PyWaPOR soil moisture calculation functions. For each parameter, we provide both the data source and the preprocessing method (if applicable).

## Table of Contents

1. [Vegetation Cover Parameters](#vegetation-cover-parameters)
2. [Temperature Extremes Parameters](#temperature-extremes-parameters)
   - [Maximum Temperature Bare Soil](#maximum-temperature-bare-soil)
   - [Maximum Temperature Full Vegetation](#maximum-temperature-full-vegetation)
3. [Core Soil Moisture Parameters](#core-soil-moisture-parameters)
4. [Intermediate Parameters and Their Calculations](#intermediate-parameters-and-their-calculations)
5. [External Data Sources](#external-data-sources)

## Vegetation Cover Parameters

### `vegetation_cover(ndvi, nd_min=0.125, nd_max=0.8, vc_pow=0.7)`

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `ndvi` | **Sentinel-2 MSI bands** (10m resolution)<br>Calculate as: (NIR - RED) / (NIR + RED) | Quality filtering based on cloud masks<br>Time-compositing if needed |
| `nd_min` | **Default value (0.125)** or calibrate based on local conditions | Calibration requires ground truth vegetation cover measurements |
| `nd_max` | **Default value (0.8)** or calibrate based on local conditions | Calibration requires ground truth vegetation cover measurements |
| `vc_pow` | **Default value (0.7)** | This shape parameter can be adjusted based on vegetation type and density validation data |

## Temperature Extremes Parameters

### Maximum Temperature Bare Soil

#### `maximum_temperature_bare(ra_hor_clear_i, emiss_atm_i, t_air_k_i, ad_i, raa, ras, r0_bare=0.38)`

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `ra_hor_clear_i` | **ERA5-Land or ground station data**<br>Incoming solar radiation [W m^-2] | Temporal interpolation to satellite overpass time<br>Spatial interpolation to match study area grid |
| `emiss_atm_i` | **Calculated from meteorological data**<br>Using `atmospheric_emissivity()` function | Requires air temperature, vapor pressure, and atmospheric pressure |
| `t_air_k_i` | **ERA5-Land or ground station data**<br>Air temperature [K] | Convert from °C to Kelvin if needed<br>Temporal and spatial interpolation |
| `ad_i` | **Calculated from meteorological data**<br>Using `air_density()` function | Requires air temperature, specific humidity, and atmospheric pressure |
| `raa` | **Calculated from meteorological data**<br>Using `aerodynamic_resistance_bare()` function | Requires wind speed, friction velocity, and Monin-Obukhov length |
| `ras` | **Calculated using soil properties**<br>Using `soil_resistance()` function | Typically uses a constant value of 10 s/m for operational purposes |
| `r0_bare` | **Default value (0.38)** or spectral measurements | Can be obtained from Landsat/Sentinel surface reflectance products |

### Maximum Temperature Full Vegetation

#### `maximum_temperature_full(ra_hor_clear_i, emiss_atm_i, t_air_k_i, ad_i, rac, r0_full=0.18)`

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `ra_hor_clear_i` | **ERA5-Land or ground station data**<br>Incoming solar radiation [W m^-2] | Same as for bare soil function |
| `emiss_atm_i` | **Calculated from meteorological data**<br>Using `atmospheric_emissivity()` function | Same as for bare soil function |
| `t_air_k_i` | **ERA5-Land or ground station data**<br>Air temperature [K] | Same as for bare soil function |
| `ad_i` | **Calculated from meteorological data**<br>Using `air_density()` function | Same as for bare soil function |
| `rac` | **Calculated from vegetation properties**<br>Using `aerodynamic_resistance_full()` function | Requires wind speed, vegetation height, and atmospheric stability corrections |
| `r0_full` | **Default value (0.18)** or spectral measurements | Can be calibrated based on vegetation type |

### Maximum Temperature Combined

#### `maximum_temperature(t_max_bare, t_max_full, vc)`

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `t_max_bare` | **Output from `maximum_temperature_bare()`** | Direct use of function output |
| `t_max_full` | **Output from `maximum_temperature_full()`** | Direct use of function output |
| `vc` | **Output from `vegetation_cover()`** | Direct use of function output |

## Core Soil Moisture Parameters

### `soil_moisture_from_maximum_temperature(lst_max, lst, lst_min)`

| Parameter | Source | Preprocessing |
|-----------|--------|--------------|
| `lst_max` | **Output from `maximum_temperature()`**<br>Maximum temperature under dry conditions [K] | Direct use of function output |
| `lst` | **Landsat 8/9 TIRS bands** or **MODIS LST product**<br>Actual Land Surface Temperature [K] | Atmospheric correction<br>Emissivity correction<br>Quality filtering<br>Sharpening to 10m (if using MODIS or similar coarser data) |
| `lst_min` | **Calculated from wet bulb temperature**<br>Using `minimum_temperature()` function | Requires air temperature, vegetation cover, and wet bulb temperature |

## Intermediate Parameters and Their Calculations

### Aerodynamic Resistances

| Parameter | Calculation | Input Sources |
|-----------|------------|---------------|
| `raa` | `aerodynamic_resistance_bare(ustar, h_obs, disp=0, z0m=0.0025, L=None)` | Wind speed measurements at standard height<br>Atmospheric stability parameters |
| `ras` | Typically uses a constant value of ~10 s/m | N/A |
| `rac` | `aerodynamic_resistance_full(ustar, h_obs, h_veg, disp=None, z0m=None, L=None)` | Wind speed<br>Vegetation height (can be estimated from NDVI relationships) |

### Atmospheric Parameters

| Parameter | Calculation | Input Sources |
|-----------|------------|---------------|
| `emiss_atm_i` | `atmospheric_emissivity(t_air_k_i, vp_i, press_i)` | Air temperature [K]<br>Vapor pressure [hPa]<br>Air pressure [hPa] |
| `ad_i` | `air_density(t_air_k, q, press)` | Air temperature [K]<br>Specific humidity [kg/kg]<br>Air pressure [hPa] |
| `t_wet_k_i` | `wet_bulb_temperature(t_air_k, rh, press)` | Air temperature [K]<br>Relative humidity [%]<br>Air pressure [hPa] |

## External Data Sources

### Satellite Data

| Data Type | Satellite Source | Resolution | Access |
|-----------|-----------------|------------|--------|
| NDVI | **Sentinel-2 MSI** | 10m | [Copernicus Open Access Hub](https://scihub.copernicus.eu/) |
| Land Surface Temperature | **Landsat 8/9 TIRS** | 100m resampled | [USGS Earth Explorer](https://earthexplorer.usgs.gov/) |
| Alternative LST | **MODIS MOD11A1/MYD11A1** | 1km | [NASA LAADS DAAC](https://ladsweb.modaps.eosdis.nasa.gov/) |
| Surface Reflectance | **Sentinel-2 MSI** | 10-20m | [Copernicus Open Access Hub](https://scihub.copernicus.eu/) |

### Meteorological Data

| Data Type | Source | Resolution | Access |
|-----------|--------|------------|--------|
| Air Temperature | **ERA5-Land** | 9km | [Copernicus Climate Data Store](https://cds.climate.copernicus.eu/) |
| Vapor Pressure | **ERA5-Land** | 9km | [Copernicus Climate Data Store](https://cds.climate.copernicus.eu/) |
| Wind Speed | **ERA5-Land** | 9km | [Copernicus Climate Data Store](https://cds.climate.copernicus.eu/) |
| Solar Radiation | **ERA5-Land** | 9km | [Copernicus Climate Data Store](https://cds.climate.copernicus.eu/) |
| Alternative Source | **Local Weather Stations** | Point data | Regional meteorological services |

### Terrain Data

| Data Type | Source | Resolution | Access |
|-----------|--------|------------|--------|
| Digital Elevation Model | **SRTM** | 30m | [USGS Earth Explorer](https://earthexplorer.usgs.gov/) |
| Alternative DEM | **ALOS PALSAR** | 12.5m | [Alaska Satellite Facility](https://asf.alaska.edu/) |

## Preprocessing Workflow Example

To prepare all inputs for soil moisture calculation:

1. **Download raw data**:
   - Sentinel-2 L1C images (for NDVI)
   - Landsat 8/9 imagery (for LST)
   - ERA5-Land meteorological data

2. **Process optical data**:
   - Atmospheric correction of Sentinel-2 data
   - Calculate NDVI
   - Calculate vegetation cover using `vegetation_cover()`

3. **Process thermal data**:
   - Calculate land surface temperature from Landsat thermal bands
   - Apply emissivity corrections

4. **Process meteorological data**:
   - Interpolate ERA5-Land to study area grid
   - Calculate derived parameters (air density, vapor pressure, etc.)

5. **Calculate intermediate parameters**:
   - Aerodynamic resistances
   - Maximum and minimum temperature bounds

6. **Calculate soil moisture**:
   - Apply `soil_moisture_from_maximum_temperature()`

7. **Downscaling** (if higher resolution is needed):
   - Apply thermal sharpening algorithm to produce 10m soil moisture
