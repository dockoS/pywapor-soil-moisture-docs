# Evapotranspiration Parameters: Google Earth Engine Data Sources

This document maps each parameter required for PyWaPOR evapotranspiration calculation to its corresponding Google Earth Engine (GEE) data source.

## Table of Contents
1. [Introduction](#introduction)
2. [Optical Data](#optical-data)
3. [Thermal Data](#thermal-data)
4. [Meteorological Data](#meteorological-data)
5. [Soil Data](#soil-data)
6. [Derived Parameters](#derived-parameters)

## Introduction

This guide provides specific Google Earth Engine (GEE) data sources for each parameter required in the PyWaPOR evapotranspiration calculation. For each parameter, we provide:

- Parameter name and symbol
- GEE dataset and collection
- Pre-processing steps required
- Alternative sources if available
- Any scaling or conversion factors needed

## Optical Data

### NDVI (Normalized Difference Vegetation Index)
- **GEE Collection**: `COPERNICUS/S2_SR` (Sentinel-2) or `LANDSAT/LC09/C02/T1_L2` (Landsat 9)
- **Calculation**: `image.normalizedDifference(['B8', 'B4'])` for Sentinel-2
- **Temporal Resolution**: 10-day composites recommended
- **Spatial Resolution**: 10m (Sentinel-2), 30m (Landsat)
- **PyWaPOR Usage**: Input for vegetation cover calculation
- **GEE Code Example**:
  ```javascript
  var ndvi = s2Image.normalizedDifference(['B8', 'B4']);
  ```

### Albedo (Surface reflectance)
- **GEE Collection**: `COPERNICUS/S2_SR`
- **Calculation**: Weighted sum of bands (B2×0.175 + B3×0.227 + B4×0.233 + B8×0.365)
- **Spatial Resolution**: 10m
- **PyWaPOR Usage**: Used for radiation balance calculation
- **GEE Code Example**:
  ```javascript
  var albedo = s2Image.select('B2').multiply(0.175)
              .add(s2Image.select('B3').multiply(0.227))
              .add(s2Image.select('B4').multiply(0.233))
              .add(s2Image.select('B8').multiply(0.365));
  ```

### LAI (Leaf Area Index)
- **GEE Source**: Derived from NDVI and vegetation cover using PyWaPOR formula
- **Calculation**: `lai = (1-vc).log().divide(-0.45)` where lai_pow = -0.45
- **Spatial Resolution**: Same as NDVI (10m)
- **PyWaPOR Usage**: Used for interception, transpiration calculations
- **GEE Code Example**:
  ```javascript
  var lai = ee.Image(1).subtract(vc).log().divide(-0.45);
  ```

### Vegetation Cover
- **GEE Source**: Derived from NDVI
- **Calculation**: Using PyWaPOR formula with nd_min=0.125, nd_max=0.8, vc_pow=0.7
- **Formula**: `vc = 1 - ((nd_max - ndvi) / (nd_max - nd_min)) ^ vc_pow`
- **PyWaPOR Usage**: Critical parameter for partitioning between soil and vegetation
- **GEE Code Example**:
  ```javascript
  var nd_min = 0.125;
  var nd_max = 0.8;
  var vc_pow = 0.7;
  var vc_min = 0;
  var vc_max = 0.9677324224821418;
  
  var vc = ee.Image(1).subtract(
    ndvi.subtract(nd_min).multiply(-1)
    .add(nd_max.subtract(nd_min))
    .divide(nd_max.subtract(nd_min))
    .pow(vc_pow)
  ).clamp(vc_min, vc_max);
  ```

## Thermal Data

### LST (Land Surface Temperature)
- **GEE Collection**: `LANDSAT/LC09/C02/T1_L2` (ST_B10 band)
- **Processing**: Scale factor correction: `lst = lst.multiply(0.00341802).add(149.0).subtract(273.15);` (converts to °C)
- **Spatial Resolution**: 30m (native), can be downscaled to 10m
- **Temporal Resolution**: 16 days
- **PyWaPOR Usage**: Critical parameter for energy balance and soil moisture calculation
- **Downscaling**: Can use Decision Tree Sharpener (DMS) with NDVI to downscale to 10m
- **GEE Code Example**:
  ```javascript
  var lst = landsat.select('ST_B10').multiply(0.00341802).add(149.0).subtract(273.15);
  ```

### Emissivity
- **GEE Source**: Derived from vegetation cover
- **Formula**: `emiss = emiss_bare + vc × (emiss_full - emiss_bare)`
- **Parameters**: emiss_bare = 0.95, emiss_full = 0.99
- **PyWaPOR Usage**: Required for radiation balance calculations
- **GEE Code Example**:
  ```javascript
  var emiss_bare = 0.95;
  var emiss_full = 0.99;
  var emissivity = ee.Image(emiss_bare).add(vc.multiply(emiss_full - emiss_bare));
  ```

## Meteorological Data

### Air Temperature
- **GEE Collection**: `ECMWF/ERA5_LAND/DAILY`
- **Band**: 'temperature_2m'
- **Processing**: Convert from K to °C: `t_air = t_air.subtract(273.15);`
- **Spatial Resolution**: ~9km (0.1°)
- **Temporal Resolution**: Daily
- **PyWaPOR Usage**: Used for calculating stress_temp and other energy balance parameters
- **GEE Code Example**:
  ```javascript
  var t_air = era5.select('temperature_2m').subtract(273.15);
  ```

### Wind Speed
- **GEE Collection**: `ECMWF/ERA5_LAND/DAILY`
- **Bands**: 'u_component_of_wind_10m', 'v_component_of_wind_10m'
- **Calculation**: `wind_speed = sqrt(u_10m^2 + v_10m^2)`
- **Spatial Resolution**: ~9km (0.1°)
- **PyWaPOR Usage**: Required for aerodynamic resistance calculation
- **GEE Code Example**:
  ```javascript
  var u_10m = era5.select('u_component_of_wind_10m');
  var v_10m = era5.select('v_component_of_wind_10m');
  var wind_speed = u_10m.multiply(u_10m).add(v_10m.multiply(v_10m)).sqrt();
  ```

### Solar Radiation
- **GEE Collection**: `ECMWF/ERA5_LAND/DAILY`
- **Band**: 'surface_solar_radiation_downwards_daily'
- **Processing**: Convert from J/m²/day to W/m²: `ra_24 = ra_24.divide(86400);`
- **Spatial Resolution**: ~9km (0.1°)
- **PyWaPOR Usage**: Used for calculating radiation stress and net radiation
- **GEE Code Example**:
  ```javascript
  var ra_24 = era5.select('surface_solar_radiation_downwards_daily').divide(86400);
  ```

### Vapor Pressure and VPD
- **GEE Collection**: `ECMWF/ERA5_LAND/DAILY`
- **Bands**: 'temperature_2m', 'dewpoint_temperature_2m'
- **Calculation for Vapor Pressure**: From dewpoint temperature using [Magnus-Tetens formula](https://en.wikipedia.org/wiki/Vapour_pressure_of_water)
- **Calculation for VPD**: `svp_24 - vp_24` where svp_24 is saturation vapor pressure
- **PyWaPOR Usage**: Essential for stress_vpd calculation and ET reference
- **GEE Code Example**:
  ```javascript
  var dewpoint = era5.select('dewpoint_temperature_2m').subtract(273.15); // Convert to °C
  var vp_24 = dewpoint.expression('0.6108 * exp(17.27 * b() / (b() + 237.3))', {}).multiply(10); // mbar
  
  var t_air_c = era5.select('temperature_2m').subtract(273.15); // Convert to °C
  var svp_24 = t_air_c.expression('0.6108 * exp(17.27 * b() / (b() + 237.3))', {}).multiply(10); // mbar
  
  var vpd_24 = svp_24.subtract(vp_24);
  ```

### Precipitation
- **GEE Collection**: Primary: `UCSB-CHG/CHIRPS/DAILY` (5km)
- **Alternative**: `ECMWF/ERA5_LAND/DAILY` band 'total_precipitation'
- **Processing for ERA5**: Convert from m to mm: `P_24 = P_24.multiply(1000);`
- **Spatial Resolution**: 5km (CHIRPS), ~9km (ERA5)
- **PyWaPOR Usage**: Required for interception calculation
- **GEE Code Example**:
  ```javascript
  // CHIRPS (already in mm/day)
  var P_24 = chirps.select('precipitation');
  
  // OR ERA5 (convert m to mm)
  var P_24 = era5.select('total_precipitation').multiply(1000);
  ```

## Soil Data

### Soil Moisture (Top Layer)
- **GEE Source Options**:
  1. Derived from LST using PyWaPOR's `soil_moisture_from_maximum_temperature` formula
  2. `NASA_USDA/HSL/SMAP10KM_soil_moisture` collection (10km resolution)
  3. `NASA/SMAP/SPL4SMGP` collection (9km resolution)
- **PyWaPOR Usage**: Required for soil resistance and moisture stress calculations
- **Processing**: Top layer effective saturation (se_top) needs normalization between 0 and 1
- **GEE Code Example** (if derived from LST):
  ```javascript
  // Using PyWaPOR formulation with lst_max and lst_min
  var se_top = ee.Image(1).subtract(
    lst.subtract(lst_min)
    .divide(lst_max.subtract(lst_min))
  ).clamp(0, 1);
  ```

### Soil Moisture (Root Zone)
- **GEE Source Options**:
  1. Derived from top layer soil moisture with time lag
  2. `NASA/SMAP/SPL4SMGP` collection, bands 'sm_rootzone' or 'sm_rootzone_wetness'
- **PyWaPOR Usage**: Essential for moisture stress calculation
- **Calculation**: When derived from top layer, use temporal smoothing with exponential filter
- **GEE Code Example**:
  ```javascript
  // If using SMAP root zone moisture
  var se_root = smap.select('sm_rootzone_wetness'); // Already scaled 0-1
  
  // If deriving from top layer (simplified example)
  var filter = ee.Filter.date(start_date, current_date);
  var se_top_collection = ee.ImageCollection('YOUR_SE_TOP_TIMESERIES').filter(filter);
  // Apply temporal smoothing here
  var se_root = se_top_collection.mean();  // Very simplified; actual implementation would use exponential weighting
  ```

### Soil Properties
- **GEE Collections**:
  1. `OPENLANDMAP/SOL/SOL_CLAY-WFRACTION_USDA-3A1A1A_M/v02` (clay content)
  2. `OPENLANDMAP/SOL/SOL_SAND-WFRACTION_USDA-3A1A1A_M/v02` (sand content)
  3. `OPENLANDMAP/SOL/SOL_TEXTURE-CLASS_USDA-TT_M/v02` (texture class)
- **Spatial Resolution**: 250m
- **PyWaPOR Usage**: Optional; may be used to adjust soil parameters like `r_soil_min`
- **GEE Code Example**:
  ```javascript
  var clay = ee.Image('OPENLANDMAP/SOL/SOL_CLAY-WFRACTION_USDA-3A1A1A_M/v02').select('b0');
  var sand = ee.Image('OPENLANDMAP/SOL/SOL_SAND-WFRACTION_USDA-3A1A1A_M/v02').select('b0');
  // Example adjusting soil resistance based on texture
  var r_soil_min = ee.Image(800).add(clay.multiply(5));
  ```

## Derived Parameters

### Psychrometric Constant
- **Calculation**: Function of pressure and specific heat of air
- **Formula**: `psy_24 = p_air×100 × sh / (0.622 × lh_24)` 
- **Dependencies**: Pressure from digital elevation model, latent heat of vaporization
- **GEE Code Example**:
  ```javascript
  var elevation = ee.Image('USGS/SRTMGL1_003').select('elevation');
  var lapse = -0.0065;  // lapse rate K/m
  var g = 9.807;        // gravity m/s2
  var gc_spec = 287.0;  // gas constant J/kg/K
  var sh = 1004.0;      // specific heat J/kg/K
  var p0 = 1013.25;     // sea level pressure mbar
  var t0 = 20;          // sea level temperature °C
  var t0_kelvin = t0 + 273.15;
  
  // Atmospheric pressure based on elevation
  var power = g / (lapse * gc_spec);
  var p_air = ee.Image(p0).multiply(ee.Image(1).subtract(lapse.multiply(elevation).divide(t0_kelvin)).pow(power));
  
  // Latent heat of vaporization
  var lh_0 = 2501000.0; // latent heat at 0°C in J/kg
  var lh_rate = -2361;  // rate of change with temperature
  var lh_24 = ee.Image(lh_0).add(t_air.multiply(lh_rate));
  
  // Psychrometric constant
  var psy_24 = p_air.multiply(100).multiply(sh).divide(0.622 * lh_24);
  ```

### Air Density
- **Calculation**: Using temperature and pressure
- **Formula**: `ad_24 = p_air×100 / (gc_spec×T_air_K)`
- **Dependencies**: Air temperature, atmospheric pressure
- **GEE Code Example**:
  ```javascript
  var t_air_k = t_air.add(273.15);  // Convert to Kelvin
  var ad_24 = p_air.multiply(100).divide(gc_spec.multiply(t_air_k));
  ```

### Stress Factors
- **Radiation Stress**:
  ```javascript
  var stress_rad = ra_24.divide(ra_24.add(60)).multiply(ee.Image(1).add(ee.Image(60).divide(500)));
  stress_rad = stress_rad.clamp(0, 1);
  ```

- **Temperature Stress**:
  ```javascript
  var t_opt = 25.0;  // optimum temperature for plant growth
  var t_min = 0.0;   // minimum temperature for plant growth
  var t_max = 50.0;  // maximum temperature for plant growth
  
  var f = ee.Image(t_max - t_opt).divide(t_opt - t_min);
  var x = t_air.subtract(t_min).multiply(t_max.subtract(t_air).pow(f));
  var y = ee.Image(t_opt - t_min).multiply(ee.Image(t_max - t_opt).pow(f));
  var stress_temp = x.divide(y).clamp(0, 1);
  ```

- **Moisture Stress**:
  ```javascript
  var tenacity = 1.5;  // for moderately sensitive plants
  var stress_moist = se_root.multiply(tenacity)
    .subtract(ee.Image(1).divide(ee.Image(2).multiply(Math.PI))
    .multiply(se_root.multiply(2).multiply(Math.PI).sin()));
  stress_moist = stress_moist.clamp(0, 1);
  ```

- **VPD Stress**:
  ```javascript
  var vpd_slope = -0.3;  // curve slope for VPD stress
  var stress_vpd = vpd_24.multiply(0.1).add(0.5).log().multiply(vpd_slope).add(1);
  stress_vpd = stress_vpd.clamp(0, 1);
  ```

### Roughness Parameters
- **Obstacle Height**:
  ```javascript
  var ndvi_obs_min = 0.25;
  var ndvi_obs_max = 0.75;
  var obs_fr = 0.25;
  var z_obst_max = 2.0;  // maximum obstacle height in meters
  
  var z_obst = ee.Image(z_obst_max);
  var z_obst = ee.Image().expression(
    'ndvi <= ndvi_obs_min ? obs_fr * z_obst_max : ' +
    'ndvi >= ndvi_obs_max ? z_obst_max : ' +
    '(obs_fr + (1-obs_fr) * (ndvi-ndvi_obs_min)/(ndvi_obs_max-ndvi_obs_min)) * z_obst_max',
    {
      'ndvi': ndvi,
      'ndvi_obs_min': ndvi_obs_min,
      'ndvi_obs_max': ndvi_obs_max,
      'obs_fr': obs_fr,
      'z_obst_max': z_obst_max
    }
  );
  ```

### Evapotranspiration Components
- **Reference ET**:
  ```javascript
  var r_grass = 70;  // standard grass resistance [s/m]
  var ra_grass = ee.Image(208).divide(wind_speed);  // aerodynamic resistance for grass [s/m]
  
  var et_ref_24 = ssvp_24.multiply(rn_24_grass).add(ad_24.multiply(sh).multiply(vpd_24.divide(ra_grass)))
    .divide(ssvp_24.add(psy_24.multiply(ee.Image(1).add(ee.Image(r_grass).divide(ra_grass)))));
  ```

- **Convert to mm/day**:
  ```javascript
  var day_sec = 86400.0;  // seconds in a day
  var et_ref_24_mm = et_ref_24.multiply(day_sec).divide(lh_24);
  ```

- **Total Actual ET**:
  ```javascript
  var aeti_24_mm = e_24_mm.add(t_24_mm).add(int_mm);
  ```

## Implementation Considerations

### Resolution Handling
- **Resampling**: Input datasets have varying resolutions (10m to 9km)
- **Recommendation**: Resample all inputs to 10m or 30m final resolution
- **GEE Code Example**:
  ```javascript
  var target_projection = ee.Image(1).projection().atScale(30);
  var resampled_meteorological_data = era5_derived_parameter
    .reproject({
      crs: target_projection.crs(),
      scale: 30
    });
  ```

### Temporal Integration
- **Period Selection**: For consistent results, use cloud-free optical data
- **Compositing**: Create 10-day composites of optical data (NDVI, albedo)
- **Daily Parameters**: Use daily meteorological data for each 10-day period

### Validation Data Sources
- **Flux Towers**: FLUXNET or local flux tower data for validation
- **GEE Collection**: `FLUXNET/FLUXNET2015` provides ET measurements
- **Alternative**: Compare with the official PyWaPOR ET product (Level 1, 2, or 3)
