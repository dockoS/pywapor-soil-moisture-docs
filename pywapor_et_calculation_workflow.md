# PyWaPOR Evapotranspiration Calculation: Step-by-Step Workflow

This document outlines the complete step-by-step workflow for implementing the PyWaPOR evapotranspiration calculation using Google Earth Engine. It describes all parameters, their sources, outputs, and how they interact throughout the calculation process.

## Table of Contents
1. [Data Collection](#data-collection)
2. [Preprocessing](#preprocessing)
3. [Vegetation Parameters](#vegetation-parameters)
4. [Surface Properties](#surface-properties)
5. [Meteorological Processing](#meteorological-processing)
6. [Moisture Parameters](#moisture-parameters)
7. [Resistance Calculation](#resistance-calculation)
8. [Stress Factor Calculation](#stress-factor-calculation)
9. [Energy Balance Components](#energy-balance-components)
10. [Evaporation and Transpiration](#evaporation-and-transpiration)
11. [Final ET Calculation](#final-et-calculation)
12. [Post-processing and Validation](#post-processing-and-validation)

## Data Collection

### Step 1: Define Area of Interest and Time Period
- **Input**: GEE Geometry (polygon of interest)
- **Output**: Spatial boundary for all subsequent calculations
- **Interaction**: All datasets will be filtered to this geometry

### Step 2: Acquire Optical Data
- **Source**: Sentinel-2 Level 2A surface reflectance (10m resolution)
- **Parameters**: Bands 2, 3, 4, 8 (Blue, Green, Red, NIR)
- **Output**: Cloud-free image or composite for the time period
- **Interaction**: Input for vegetation indices, albedo

### Step 3: Acquire Thermal Data
- **Source**: Landsat 8/9 Surface Temperature (30m resolution)
- **Parameters**: ST_B10 (Surface Temperature)
- **Output**: Land surface temperature in Celsius
- **Interaction**: Critical input for energy balance, soil moisture estimation

### Step 4: Acquire Meteorological Data
- **Source**: ERA5-Land daily aggregates (9km resolution)
- **Parameters**: Temperature, wind components, radiation, dewpoint temperature, precipitation
- **Output**: Meteorological parameters at daily timescale
- **Interaction**: Essential for energy balance, atmospheric conditions

### Step 5: Acquire DEM Data
- **Source**: SRTM Digital Elevation Model
- **Parameters**: Elevation
- **Output**: Surface elevation in meters
- **Interaction**: Used for pressure calculation, which affects psychrometric constant

## Preprocessing

### Step 6: Create Cloud-Free Optical Composite
- **Input**: Sentinel-2 collection
- **Process**: Filter by cloud percentage, create temporal composite
- **Output**: Single cloud-free image with bands B2, B3, B4, B8
- **Interaction**: Required for reliable vegetation index calculation

### Step 7: Resample Thermal Data
- **Input**: Landsat surface temperature
- **Process**: Convert to Celsius, align to study area
- **Output**: LST in degrees Celsius
- **Interaction**: Used directly in energy balance and soil moisture

### Step 8: Process Meteorological Data
- **Input**: ERA5-Land parameters
- **Process**: Unit conversion, resampling to target resolution
- **Output**: Meteorological parameters at consistent resolution
- **Interaction**: Inputs for multiple ET calculation steps

## Vegetation Parameters

### Step 9: Calculate NDVI
- **Input**: Sentinel-2 bands B4 (Red) and B8 (NIR)
- **Process**: (NIR - Red) / (NIR + Red)
- **Output**: NDVI raster (values from -1 to 1)
- **Interaction**: Input for vegetation cover, LAI, and roughness calculations

### Step 10: Calculate Vegetation Cover
- **Input**: NDVI, PyWaPOR constants (nd_min = 0.125, nd_max = 0.8, vc_pow = 0.7)
- **Process**: Use PyWaPOR formula: 1 - ((nd_max - NDVI) / (nd_max - nd_min))^vc_pow
- **Output**: Vegetation cover fraction (0 to 0.968)
- **Interaction**: Critical parameter for partitioning between soil and vegetation

### Step 11: Calculate Leaf Area Index (LAI)
- **Input**: Vegetation cover, PyWaPOR constant (lai_pow = -0.45)
- **Process**: LAI = log(1 - vc) / (-0.45)
- **Output**: LAI values typically 0 to 5+
- **Interaction**: Used for interception, aerodynamic resistance, canopy resistance

## Surface Properties

### Step 12: Calculate Albedo
- **Input**: Sentinel-2 bands (B2, B3, B4, B8)
- **Process**: Weighted sum (B2×0.175 + B3×0.227 + B4×0.233 + B8×0.365)
- **Output**: Surface albedo (typical range 0.1 to 0.4)
- **Interaction**: Used in radiation balance calculations

### Step 13: Calculate Emissivity
- **Input**: Vegetation cover, emissivity constants (emiss_bare = 0.95, emiss_full = 0.99)
- **Process**: emissivity = emiss_bare + vc × (emiss_full - emiss_bare)
- **Output**: Surface emissivity (0.95 to 0.99)
- **Interaction**: Used in radiation balance calculations

### Step 14: Calculate Surface Roughness Parameters
- **Input**: NDVI, LAI, vegetation parameters (ndvi_obs_min, ndvi_obs_max, obs_fr, z_obst_max)
- **Process**: 
  1. Calculate obstacle height from NDVI
  2. Calculate displacement height from obstacle height and LAI
  3. Calculate roughness length
- **Output**: Roughness parameters in meters
- **Interaction**: Used for aerodynamic resistance calculation

## Meteorological Processing

### Step 15: Calculate Air Pressure from Elevation
- **Input**: Digital Elevation Model, sea level pressure, temperature
- **Process**: Barometric equation with PyWaPOR constants
- **Output**: Air pressure in mbar for each pixel
- **Interaction**: Required for psychrometric constant and air density

### Step 16: Calculate Psychrometric Constant
- **Input**: Air pressure, specific heat of air, latent heat of vaporization
- **Process**: psy_24 = p_air×100 × sh / (0.622 × lh_24)
- **Output**: Psychrometric constant (mbar K-1)
- **Interaction**: Used in Penman-Monteith equation

### Step 17: Calculate Air Density
- **Input**: Air pressure, temperature, gas constant
- **Process**: ad_24 = p_air×100 / (gc_spec×T_air_K)
- **Output**: Air density (kg m-3)
- **Interaction**: Used in energy transfer calculations

### Step 18: Calculate Vapor Pressure Parameters
- **Input**: Temperature, dewpoint temperature
- **Process**: 
  1. Calculate actual vapor pressure from dewpoint
  2. Calculate saturation vapor pressure from temperature
  3. Calculate vapor pressure deficit (VPD)
- **Output**: Vapor pressure parameters (mbar)
- **Interaction**: Used for ET reference and stress calculations

### Step 19: Calculate Slope of Saturation Vapor Pressure Curve
- **Input**: Air temperature
- **Process**: Use temperature-dependent formula
- **Output**: Slope value (mbar K-1)
- **Interaction**: Used in Penman-Monteith equation

## Moisture Parameters

### Step 20: Calculate Temperature Extremes for Soil Moisture
- **Input**: LST, vegetation cover, radiation parameters
- **Process**: 
  1. Calculate maximum temperature for bare soil
  2. Calculate maximum temperature for full vegetation
  3. Calculate minimum temperature
- **Output**: Temperature bounds for each pixel
- **Interaction**: Used for soil moisture estimation

### Step 21: Estimate Top Layer Soil Moisture
- **Input**: LST, temperature extremes
- **Process**: Normalize and invert LST between wet and dry limits
- **Output**: Relative soil moisture (0-1)
- **Interaction**: Input for soil resistance, stress calculation

### Step 22: Estimate Root Zone Soil Moisture
- **Input**: Top layer soil moisture, time series data
- **Process**: Apply temporal filter to account for deeper moisture dynamics
- **Output**: Root zone relative soil moisture (0-1)
- **Interaction**: Critical input for moisture stress calculation

## Stress Factor Calculation

### Step 23: Calculate Radiation Stress
- **Input**: Solar radiation, PyWaPOR constants
- **Process**: stress_rad = ra_24/(ra_24 + 60) × (1 + 60/500)
- **Output**: Radiation stress factor (0-1)
- **Interaction**: Component of canopy resistance

### Step 24: Calculate Temperature Stress
- **Input**: Air temperature, temperature thresholds (t_opt=25°C, t_min=0°C, t_max=50°C)
- **Process**: Use PyWaPOR formula with asymmetric response curve
- **Output**: Temperature stress factor (0-1)
- **Interaction**: Component of canopy resistance

### Step 25: Calculate Moisture Stress
- **Input**: Root zone soil moisture, tenacity factor (1.5)
- **Process**: stress_moist = tenacity×se_root - sin(2π×se_root)/(2π)
- **Output**: Moisture stress factor (0-1)
- **Interaction**: Component of canopy resistance

### Step 26: Calculate VPD Stress
- **Input**: Vapor pressure deficit, vpd_slope (-0.3)
- **Process**: stress_vpd = vpd_slope×ln(0.1×vpd_24+0.5) + 1
- **Output**: VPD stress factor (0-1)
- **Interaction**: Component of canopy resistance

## Resistance Calculation

### Step 27: Calculate Aerodynamic Resistance
- **Input**: Wind speed, roughness parameters
- **Process**: Use logarithmic wind profile equation
- **Output**: Aerodynamic resistance (s m-1)
- **Interaction**: Used in energy transfer calculations

### Step 28: Calculate Soil Resistance
- **Input**: Top layer soil moisture, soil resistance parameters
- **Process**: r_soil = r_soil_min × se_top^r_soil_pow
- **Output**: Soil resistance (s m-1)
- **Interaction**: Controls soil evaporation rate

### Step 29: Calculate Atmospheric Canopy Resistance
- **Input**: LAI, stress factors (radiation, temperature, VPD)
- **Process**: r_canopy_0 = (rs_min / lai_eff) / (stress_rad × stress_temp × stress_vpd)
- **Output**: Atmospheric canopy resistance (s m-1)
- **Interaction**: Intermediate for full canopy resistance

### Step 30: Calculate Full Canopy Resistance
- **Input**: Atmospheric canopy resistance, moisture stress
- **Process**: r_canopy = r_canopy_0 / stress_moist
- **Output**: Canopy resistance (s m-1)
- **Interaction**: Controls transpiration rate

## Energy Balance Components

### Step 31: Calculate Net Radiation
- **Input**: Solar radiation, albedo, LST, emissivity, air temperature
- **Process**: 
  1. Calculate shortwave net radiation (1-albedo) × solar_radiation
  2. Calculate longwave radiation components
  3. Sum to get total net radiation
- **Output**: Net radiation (W m-2)
- **Interaction**: Primary energy input for evapotranspiration

### Step 32: Calculate Soil Heat Flux
- **Input**: Net radiation, vegetation cover
- **Process**: G = Rn × (0.05 + 0.18×exp(-0.521×LAI))
- **Output**: Soil heat flux (W m-2)
- **Interaction**: Energy not available for evapotranspiration

### Step 33: Calculate Available Energy
- **Input**: Net radiation, soil heat flux
- **Process**: Available energy = Rn - G
- **Output**: Available energy (W m-2)
- **Interaction**: Energy available for evaporation and transpiration

## Evaporation and Transpiration

### Step 34: Calculate Interception
- **Input**: Precipitation, vegetation cover, LAI, maximum interception parameter
- **Process**: int_mm = int_max × lai × (1 - (1 / (1 + ((vc × P_24) / (int_max × lai)))))
- **Output**: Interception (mm day-1)
- **Interaction**: Component of total ET

### Step 35: Calculate Reference ET
- **Input**: Net radiation, air density, psychrometric constant, VPD, slope of saturation vapor pressure curve, wind speed
- **Process**: Penman-Monteith equation for reference grass
- **Output**: Reference ET (W m-2)
- **Interaction**: Used for comparison and quality control

### Step 36: Calculate Evaporation
- **Input**: Available energy, air density, VPD, aerodynamic resistance, soil resistance, psychrometric constant, slope of saturation vapor pressure
- **Process**: Penman-Monteith adapted for soil evaporation
- **Output**: Evaporation energy flux (W m-2)
- **Interaction**: Component of total ET

### Step 37: Calculate Transpiration
- **Input**: Available energy, air density, VPD, aerodynamic resistance, canopy resistance, psychrometric constant, slope of saturation vapor pressure
- **Process**: Penman-Monteith adapted for plant transpiration
- **Output**: Transpiration energy flux (W m-2)
- **Interaction**: Component of total ET

## Final ET Calculation

### Step 38: Convert Energy Fluxes to Water Depth
- **Input**: Evaporation flux, transpiration flux, interception, latent heat of vaporization
- **Process**: Convert W m-2 to mm day-1 using latent heat and seconds per day
- **Output**: Evaporation, transpiration, and interception in mm day-1
- **Interaction**: Inputs for total ET

### Step 39: Calculate Total Actual ET
- **Input**: Evaporation, transpiration, and interception in mm day-1
- **Process**: Sum the three components
- **Output**: Actual evapotranspiration (mm day-1)
- **Interaction**: Final product

## Post-processing and Validation

### Step 40: Quality Assessment
- **Input**: Calculated ET, reference ET, precipitation
- **Process**: Check physical constraints (e.g., ET < precipitation in arid regions)
- **Output**: Quality flags for ET product
- **Interaction**: Helps identify potential calculation errors

### Step 41: Temporal Aggregation
- **Input**: Daily ET values
- **Process**: Aggregate to 10-day, monthly, or seasonal totals
- **Output**: Temporal aggregates of ET
- **Interaction**: More stable products for analysis

### Step 42: Validation
- **Input**: Calculated ET, reference datasets (flux towers, PyWaPOR official product)
- **Process**: Statistical comparison
- **Output**: Validation metrics (RMSE, bias, r²)
- **Interaction**: Confirms reliability of implementation

### Step 43: Documentation and Reporting
- **Input**: All calculation steps and intermediate outputs
- **Process**: Compile documentation on methodology and results
- **Output**: Technical report
- **Interaction**: Ensures reproducibility and transparency of results
