# Technical Implementation of Relative Soil Moisture in PyWaPOR Level 3

## Table of Contents
1. [Introduction](#introduction)
2. [Theoretical Framework](#theoretical-framework)
3. [Implementation in PyWaPOR](#implementation-in-pywapor)
4. [Input Data Sources](#input-data-sources)
5. [Processing Chain](#processing-chain)
6. [Downscaling Process](#downscaling-process)
7. [Code Details](#code-details)
8. [Validation and Limitations](#validation-and-limitations)
9. [References](#references)

## Introduction

This document provides a detailed technical description of how relative soil moisture is calculated in the PyWaPOR library (Level 3) using the "triangle method" approach based on Land Surface Temperature (LST) data and downscaled to 10-meter resolution using optical imagery. The PyWaPOR package implements the SERoot (Surface Energy Balance) model to estimate relative soil moisture.

## Theoretical Framework

### The Triangle Method

The Triangle Method (or Trapezoid Method) is based on the relationship between land surface temperature and vegetation indices. This approach leverages the fact that:

1. Dry soil heats up more quickly under solar radiation (higher LST)
2. Wet soil maintains lower temperatures due to evaporative cooling (lower LST)
3. Vegetation impacts the soil temperature-moisture relationship

The model assumes that for a given vegetation cover, surface temperature will vary between two extremes:
- **Minimum temperature (T_min)**: Representing wet conditions
- **Maximum temperature (T_max)**: Representing completely dry conditions

### Relative Soil Moisture Formula

Relative soil moisture (Θ) is calculated as:

```
Θ = 1 - (T_0 - T_0,min) / (T_0,max - T_0,min)
```

Where:
- Θ (Theta): Relative root zone soil moisture (0-1 scale)
- T₀: Actual land surface temperature measured from remote sensing
- T₀,min: Minimum temperature under wet conditions
- T₀,max: Maximum temperature under dry conditions

## Implementation in PyWaPOR

### Core Components

The PyWaPOR implementation consists of two main components:

1. **SERoot Model**: Calculates relative soil moisture based on energy balance principles
2. **Thermal Sharpening**: Downscales the coarser resolution soil moisture to 10m resolution

### Key Functions and Modules

- `se_root.py`: Main driver for soil moisture calculation
- `soil_moisture.py`: Contains specific algorithms for soil moisture calculation
- `thermal_sharpener.py`: Implements thermal image downscaling
- `pyDMS.py`: Decision tree-based machine learning for downscaling

## Input Data Sources

### Level 3 Data Sources

The Level 3 PyWaPOR implementation uses the following data sources:

| Input Parameter | Source | Original Resolution | Purpose |
|-----------------|--------|---------------------|---------|
| Land Surface Temperature (LST) | Landsat 8/9 TIRS | 100m (resampled from 30m) | Primary input for temperature-based soil moisture calculation |
| NDVI | Sentinel-2 MSI | 10m | Vegetation cover calculation and downscaling |
| Green & NIR bands | Sentinel-2 MSI | 10m | Input features for downscaling |
| Atmospheric parameters | ERA5-Land | 9km | Air temperature, pressure, humidity for atmospheric corrections |
| Digital Elevation Model | SRTM | 30m | Terrain-related calculations |
| Specialized indices (NMDI, BSI, MNDWI, etc.) | Derived from Sentinel-2 | 10m | Features for thermal downscaling |

## Processing Chain

### 1. Data Preparation Phase

```
pre_se_root.main()
```

- Ingests all required input data
- Performs quality checks and filtering
- Resamples data to common grid
- Computes derived variables

### 2. Soil Moisture Calculation Phase

```
se_root.main()
```

The process follows these steps:

1. **Atmospheric Parameter Calculation**:
   - Air pressure conversion (`air_pressure_kpa2mbar`)
   - Calculation of vapor pressure, air density, etc.

2. **Vegetation Cover Estimation**:
   - NDVI-based vegetation cover calculation (`vegetation_cover`)
   - Using thresholds: nd_min=0.125, nd_max=0.8, vc_pow=0.7

3. **Radiation Budget Calculation**:
   - Longwave and shortwave radiation components
   - Net radiation for bare soil and full vegetation

4. **Aerodynamic Parameter Calculation**:
   - Wind speed adjustments
   - Friction velocities
   - Monin-Obukhov length

5. **Temperature Extremes Calculation**:
   - Maximum temperature for dry conditions (`maximum_temperature_bare`, `maximum_temperature_full`)
   - Minimum temperature for wet conditions (based on wet bulb temperature)
   - Integration based on vegetation cover (`maximum_temperature`, `minimum_temperature`)

6. **Soil Moisture Calculation**:
   - Formula implementation in `soil_moisture_from_maximum_temperature`
   - Ratio calculation: `ratio = (lst - lst_min) / (lst_max - lst_min)`
   - Clipping to 0-1 range
   - Final soil moisture: `se_root = 1 - ratio`

### 3. Downscaling Phase

```
thermal_sharpener.sharpen()
```

## Downscaling Process

### Machine Learning Approach

The PyWaPOR library uses the Decision Tree Sharpening approach to downscale low-resolution soil moisture:

1. **Training**:
   - High-resolution bands resampled to low resolution
   - Relationship established between resampled high-resolution bands and low-resolution soil moisture
   - Decision trees with linear regression at leaf nodes trained on this relationship

2. **Prediction**:
   - The trained model is applied to high-resolution bands
   - Results in a high-resolution prediction of soil moisture

### Implementation Details

**Decision Tree Model**:
```python
class DecisionTreeRegressorWithLinearLeafRegression(tree.DecisionTreeRegressor):
    # Extends the standard decision tree with linear regression at leaf nodes
    # to better capture local relationships between bands and soil moisture
```

**Key Parameters for Downscaling**:
- `vars_for_sharpening`: Default set includes ['nmdi', 'bsi', 'mndwi', 'vari_red_edge', 'psri', 'nir', 'green']
- `linearRegressionExtrapolationRatio`: 0.25 (limits extrapolation)
- `cvHomogeneityThreshold`: Used to select training pixels based on homogeneity

## Code Details

### Vegetation Cover Calculation

```python
def vegetation_cover(ndvi, nd_min=0.125, nd_max=0.8, vc_pow=0.7):
    """
    Computes vegetation cover based on NDVI.
    """
    # For NDVI ≤ nd_min: vc = 0
    # For nd_min < NDVI < nd_max: vc = 1 - ((nd_max - ndvi) / (nd_max - nd_min)) ** vc_pow
    # For NDVI ≥ nd_max: vc = 1
    
    if np.isscalar(ndvi):
        if ndvi <= nd_min:
            res = 0
        if (ndvi > nd_min) & (ndvi < nd_max):
            res = 1 - ((nd_max - ndvi) / (nd_max - nd_min)) ** vc_pow
        if ndvi >= nd_max:
            res = 1
    else:
        # Array implementation...
        
    return res
```

### Temperature Extremes Calculation

```python
def maximum_temperature_bare(ra_hor_clear_i, emiss_atm_i, t_air_k_i, ad_i, raa, ras, r0_bare=0.38):
    """
    Computes the maximum temperature under dry bare soil conditions.
    
    Parameters:
    -----------
    ra_hor_clear_i : float or array
        Incoming clear-sky horizontal shortwave radiation [W m^-2]
        This is the solar radiation reaching the surface under clear sky conditions
    
    emiss_atm_i : float or array
        Atmospheric emissivity [-]
        The effectiveness of the atmosphere in emitting thermal radiation
    
    t_air_k_i : float or array
        Air temperature [K]
        Near-surface air temperature in Kelvin
    
    ad_i : float or array
        Air density [kg m^-3]
        Mass of air per unit volume
    
    raa : float or array
        Aerodynamic resistance for heat transport from bare soil to atmosphere [s m^-1]
        Resistance against heat transfer between the soil surface and atmosphere
    
    ras : float or array
        Aerodynamic resistance for heat transport within soil boundary layer [s m^-1]
        Resistance against heat transfer in the thin layer of air immediately above soil surface
    
    r0_bare : float, default=0.38
        Albedo for dry bare soil [-]
        The fraction of incoming solar radiation reflected by dry bare soil
    """
    # The function calculates dry bare soil temperature using a complex energy balance equation
    # that accounts for net radiation, sensible heat flux, and aerodynamic resistances
```

```python
def maximum_temperature_full(ra_hor_clear_i, emiss_atm_i, t_air_k_i, ad_i, rac, r0_full=0.18):
    """
    Computes the maximum temperature under fully vegetated dry conditions.
    
    Parameters:
    -----------
    ra_hor_clear_i : float or array
        Incoming clear-sky horizontal shortwave radiation [W m^-2]
        Solar radiation reaching the surface under clear sky conditions
    
    emiss_atm_i : float or array
        Atmospheric emissivity [-]
        The effectiveness of the atmosphere in emitting thermal radiation
    
    t_air_k_i : float or array
        Air temperature [K]
        Near-surface air temperature in Kelvin
    
    ad_i : float or array
        Air density [kg m^-3]
        Mass of air per unit volume
    
    rac : float or array
        Aerodynamic resistance for heat transport from canopy to atmosphere [s m^-1]
        Resistance against heat transfer between the vegetation canopy and atmosphere
    
    r0_full : float, default=0.18
        Albedo for dry full vegetation cover [-]
        The fraction of incoming solar radiation reflected by dry vegetation
    """
    # The function calculates dry vegetation temperature using a similar energy balance approach
    # but with parameters adjusted for vegetation instead of bare soil
```

```python
def maximum_temperature(t_max_bare, t_max_full, vc):
    """
    Computes the maximum temperature at dry conditions by combining bare soil and full vegetation
    values based on vegetation cover.
    
    Parameters:
    -----------
    t_max_bare : float or array
        Maximum temperature for bare soil under dry conditions [K]
        The surface temperature of dry bare soil with no vegetation
    
    t_max_full : float or array
        Maximum temperature for full vegetation under dry conditions [K]
        The surface temperature of dry fully vegetated surfaces
    
    vc : float or array
        Vegetation cover fraction [-]
        The proportion of the pixel covered by vegetation (0-1 scale)
    
    Returns:
    --------
    t_max : float or array
        Maximum surface temperature under dry conditions [K]
        Weighted combination of bare soil and vegetation temperatures
    """
    return vc * (t_max_full - t_max_bare) + t_max_bare
```

### Core Soil Moisture Calculation

```python
def soil_moisture_from_maximum_temperature(lst_max, lst, lst_min):
    """
    Computes the relative root zone soil moisture based on estimates of
    maximum temperature and wet bulb temperature and measured land
    surface temperature.
    
    Parameters:
    -----------
    lst_max : float or array
        Maximum land surface temperature under dry conditions [K]
        This represents the theoretical upper temperature bound for completely dry soil
        at the given vegetation cover, calculated from maximum_temperature()
    
    lst : float or array
        Actual land surface temperature [K]
        Measured or estimated temperature from thermal sensors (e.g., Landsat TIRS)
        
    lst_min : float or array
        Minimum land surface temperature under wet conditions [K]
        This represents the theoretical lower temperature bound for completely wet soil
        at the given vegetation cover, calculated from minimum_temperature()
    
    Returns:
    --------
    se_root : float or array
        Relative root zone soil moisture [-]
        Value between 0 (completely dry) and 1 (field capacity/completely wet)
        Calculated as the normalized distance between actual temperature and the
        theoretical dry temperature bound
    """
    # Calculate normalized temperature ratio
    ratio = (lst - lst_min) / (lst_max - lst_min)
    
    # Clip values to 0-1 range to ensure physical consistency
    if isinstance(ratio, xr.DataArray):
        ratio = ratio.clip(0, 1)  # For xarray DataArray objects
    else:
        ratio = np.clip(ratio, 0, 1)  # For numpy arrays or scalar values
    
    # Convert temperature ratio to soil moisture (inverse relationship)
    return 1 - ratio
```

## Validation and Limitations

### Validation Approaches

- Comparison with in-situ soil moisture measurements
- Cross-validation with other satellite-based soil moisture products
- Assessment of temporal dynamics against rainfall events

### Known Limitations

1. **Vegetation Density Effects**: 
   - Dense vegetation canopies can obscure the soil moisture signal
   - Performance decreases with vegetation cover above certain thresholds

2. **Soil Type Dependency**:
   - Different soil types have different thermal properties
   - Model assumes uniform soil properties within the study area

3. **Cloudy Conditions**:
   - Optical and thermal data require clear-sky conditions
   - Results in data gaps during cloudy periods

4. **Downscaling Artifacts**:
   - Potential artifacts at boundaries between different land cover types
   - Validation recommended for very heterogeneous landscapes

## References

1. Yang, Y., et al. (2015). "Estimation of Surface Soil Moisture from Thermal Infrared Remote Sensing Using an Improved Trapezoid Method." Remote Sensing, 7(7), 8250-8270.

2. Gao, F., et al. (2012). "Toward mapping surface energy fluxes at field scales using MODIS and Landsat data." Remote Sensing of Environment, 112, 2489-2503.

3. Brutsaert, W. (1999). "Aspects of bulk atmospheric boundary layer similarity under free-convective conditions." Reviews of Geophysics, 37(4), 439-451.

4. Stull, R. (2011). "Wet-bulb temperature from relative humidity and air temperature." Journal of Applied Meteorology and Climatology, 50(11), 2267-2269.

5. FAO (2020). "WaPOR: The FAO portal to monitor Water Productivity through Open access of Remotely sensed derived data." Food and Agriculture Organization of the United Nations.

6. Brutsaert, W. (1975). "On a derivable formula for long‐wave radiation from clear skies." Water Resources Research, 11(5), 742-744.
