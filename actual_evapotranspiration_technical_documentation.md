# Technical Implementation of Actual Evapotranspiration in PyWaPOR Level 3

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

This document provides a detailed technical description of how actual evapotranspiration (ET) is calculated in the PyWaPOR library (Level 3) using a modified ETLook model. The implementation combines energy balance principles with optical and thermal remote sensing data to estimate actual evapotranspiration at high spatial resolution (10m).

## Theoretical Framework

### Two-Source Energy Balance Model

PyWaPOR implements a two-source energy balance model that separately treats soil evaporation and plant transpiration. The model is based on the ETLook framework which solves the energy balance equation:

```
LE = Rn - G - H
```

Where:
- LE: Latent heat flux (energy used for evapotranspiration)
- Rn: Net radiation
- G: Soil heat flux
- H: Sensible heat flux

The model distinguishes between:
- **LEi**: Interception evaporation
- **LEe**: Soil evaporation
- **LEt**: Transpiration

With the total actual ET being:
```
ET_act = LEi + LEe + LEt
```

### Penman-Monteith Approach

For calculating the actual evapotranspiration components, PyWaPOR uses a modified Penman-Monteith equation:

```
LE = (Δ × (Rn - G) + ρ × cp × (es - ea) / ra) / (Δ + γ × (1 + rs/ra))
```

Where:
- Δ: Slope of the saturation vapor pressure curve
- ρ: Air density
- cp: Specific heat of air at constant pressure
- es: Saturation vapor pressure
- ea: Actual vapor pressure
- ra: Aerodynamic resistance
- rs: Surface resistance
- γ: Psychrometric constant

## Implementation in PyWaPOR

### Core Components

The PyWaPOR implementation consists of several modules:

1. **ETLook Model**: Core evapotranspiration calculation
2. **Energy Balance Components**: Calculation of radiation, heat fluxes
3. **Resistance Models**: Aerodynamic and surface resistances
4. **Downscaling**: High-resolution ET estimation

### Key Modules and Functions

- `et_look.py`: Main driver for ET calculation
- `radiation.py`: Net radiation calculations
- `meteo.py`: Meteorological parameter processing
- `evapotranspiration.py`: Core ET component calculations
- `resistances.py`: Aerodynamic and surface resistance calculations

## Input Data Sources

### Level 3 Data Sources

| Input Parameter | Source | Original Resolution | Purpose |
|-----------------|--------|---------------------|---------|
| Land Surface Temperature (LST) | Landsat 8/9 TIRS | 100m (resampled) | Energy balance calculations |
| NDVI | Sentinel-2 MSI | 10m | Vegetation parameters and downscaling |
| Albedo | Sentinel-2 MSI | 10m | Radiation balance |
| LAI | Derived from Sentinel-2 | 10m | Canopy resistance |
| Atmospheric parameters | ERA5-Land | 9km | Air temperature, pressure, humidity |
| Wind speed | ERA5-Land | 9km | Aerodynamic resistance |
| Solar radiation | ERA5-Land | 9km | Radiation balance |
| Precipitation | CHIRPS or similar | 5km | Interception calculation |
| Digital Elevation Model | SRTM | 30m | Terrain corrections |

## Processing Chain

### 1. Data Preparation Phase

```python
pre_et_look.main()
```

- Ingests all required input data
- Performs quality checks and filtering
- Resamples data to common grid
- Computes derived variables

### 2. Evapotranspiration Calculation Phase

```python
et_look.main()
```

The process follows these steps:

1. **Radiation Balance Calculation**:
   - Net shortwave radiation calculation using albedo
   - Net longwave radiation calculation using LST and emissivity
   - Total net radiation for soil and vegetation components

2. **Soil Heat Flux Calculation**:
   - Based on net radiation and vegetation cover
   - Adjusted for soil moisture conditions

3. **Resistance Calculations**:
   - Aerodynamic resistance for momentum transfer
   - Aerodynamic resistance for heat transfer
   - Soil surface resistance based on soil moisture
   - Canopy resistance based on vegetation parameters and stress factors

4. **Evaporation and Transpiration Components**:
   - Soil evaporation calculation
   - Plant transpiration calculation
   - Interception evaporation calculation

5. **Total ET Calculation**:
   - Combination of all ET components
   - Conversion to appropriate units (mm/day)

### 3. Downscaling Phase

```python
thermal_sharpener.sharpen()
```

## Downscaling Process

The downscaling approach for ET is similar to that used for soil moisture:

1. **Machine Learning Based Downscaling**:
   - Uses high-resolution optical bands to downscale coarser ET estimates
   - Decision tree regression with multiple predictor variables

2. **ET-Specific Considerations**:
   - Vegetation indices are strongly correlated with ET patterns
   - Surface temperature sharpening improves ET component estimation

## Code Details

### Net Radiation Calculation

```python
def net_radiation(albedo, ra_hor_clear, emiss_s, lst_k, emiss_atm, t_air_k):
    """
    Computes net radiation at the surface.
    
    Parameters:
    -----------
    albedo : float or array
        Surface albedo [-]
        Fraction of incoming solar radiation reflected by the surface
        
    ra_hor_clear : float or array
        Incoming clear-sky horizontal shortwave radiation [W m^-2]
        Solar radiation reaching the surface under clear sky conditions
        
    emiss_s : float or array
        Surface emissivity [-]
        The effectiveness of the surface in emitting thermal radiation
        
    lst_k : float or array
        Land surface temperature [K]
        Surface temperature derived from thermal imagery
        
    emiss_atm : float or array
        Atmospheric emissivity [-]
        The effectiveness of the atmosphere in emitting thermal radiation
        
    t_air_k : float or array
        Air temperature [K]
        Near-surface air temperature in Kelvin
        
    Returns:
    --------
    rn : float or array
        Net radiation at the surface [W m^-2]
        The available energy for evapotranspiration, sensible heat, and soil heat
    """
    # Net shortwave radiation
    r_ns = (1 - albedo) * ra_hor_clear
    
    # Net longwave radiation
    r_nl = emiss_s * (emiss_atm * sb_const * t_air_k**4 - sb_const * lst_k**4)
    
    # Total net radiation
    rn = r_ns + r_nl
    
    return rn
```

### Soil Evaporation Calculation

```python
def soil_evaporation(rn_soil, g_soil, ad_i, vp_s, vp_a, raa_e, rss):
    """
    Computes the soil evaporation using the Penman-Monteith equation.
    
    Parameters:
    -----------
    rn_soil : float or array
        Net radiation available at the soil surface [W m^-2]
        Energy available for soil processes
        
    g_soil : float or array
        Soil heat flux [W m^-2]
        Energy stored in the soil
        
    ad_i : float or array
        Air density [kg m^-3]
        Mass of air per unit volume
        
    vp_s : float or array
        Saturation vapor pressure at the soil surface [kPa]
        Maximum vapor pressure possible at the soil temperature
        
    vp_a : float or array
        Actual vapor pressure in the air [kPa]
        Actual water vapor content of the air
        
    raa_e : float or array
        Aerodynamic resistance for evaporation [s m^-1]
        Resistance against water vapor transfer from soil to atmosphere
        
    rss : float or array
        Soil surface resistance [s m^-1]
        Resistance against evaporation from within the soil
        
    Returns:
    --------
    le_soil : float or array
        Latent heat flux from soil evaporation [W m^-2]
        Energy used for evaporation from the soil surface
    """
    # Calculate slope of saturation vapor pressure curve
    delta_svp = calc_delta_sat_vp(t_soil_k)
    
    # Calculate psychrometric constant
    gamma = psychrometric_constant(pressure)
    
    # Penman-Monteith equation for soil
    numerator = delta_svp * (rn_soil - g_soil) + ad_i * c_p * (vp_s - vp_a) / raa_e
    denominator = delta_svp + gamma * (1 + rss / raa_e)
    
    le_soil = numerator / denominator
    
    return le_soil
```

### Transpiration Calculation

```python
def transpiration(rn_canopy, ad_i, vp_s, vp_a, raa_t, rst):
    """
    Computes the transpiration using the Penman-Monteith equation.
    
    Parameters:
    -----------
    rn_canopy : float or array
        Net radiation intercepted by the canopy [W m^-2]
        Energy available for canopy processes
        
    ad_i : float or array
        Air density [kg m^-3]
        Mass of air per unit volume
        
    vp_s : float or array
        Saturation vapor pressure at the leaf temperature [kPa]
        Maximum vapor pressure possible at the leaf temperature
        
    vp_a : float or array
        Actual vapor pressure in the air [kPa]
        Actual water vapor content of the air
        
    raa_t : float or array
        Aerodynamic resistance for transpiration [s m^-1]
        Resistance against water vapor transfer from leaves to atmosphere
        
    rst : float or array
        Stomatal resistance [s m^-1]
        Resistance against water vapor flow through leaf stomata
        
    Returns:
    --------
    le_transpiration : float or array
        Latent heat flux from transpiration [W m^-2]
        Energy used for transpiration from the vegetation
    """
    # Calculate slope of saturation vapor pressure curve
    delta_svp = calc_delta_sat_vp(t_leaf_k)
    
    # Calculate psychrometric constant
    gamma = psychrometric_constant(pressure)
    
    # Penman-Monteith equation for canopy
    numerator = delta_svp * rn_canopy + ad_i * c_p * (vp_s - vp_a) / raa_t
    denominator = delta_svp + gamma * (1 + rst / raa_t)
    
    le_transpiration = numerator / denominator
    
    return le_transpiration
```

### Total Evapotranspiration Calculation

```python
def total_evapotranspiration(le_soil, le_transpiration, le_interception):
    """
    Computes the total actual evapotranspiration by combining soil evaporation, 
    transpiration, and interception.
    
    Parameters:
    -----------
    le_soil : float or array
        Latent heat flux from soil evaporation [W m^-2]
        Energy used for evaporation from the soil surface
        
    le_transpiration : float or array
        Latent heat flux from transpiration [W m^-2]
        Energy used for transpiration from the vegetation
        
    le_interception : float or array
        Latent heat flux from interception evaporation [W m^-2]
        Energy used for evaporation of intercepted precipitation
        
    Returns:
    --------
    et_actual : float or array
        Total actual evapotranspiration [mm day^-1]
        Total water loss from surface to atmosphere
    """
    # Total latent heat flux
    le_total = le_soil + le_transpiration + le_interception
    
    # Convert from energy units to water depth
    # Factor 86400 converts from seconds to day
    et_actual = le_total * 86400 / (2.45e6)
    
    return et_actual
```

### Aerodynamic Resistance Calculation

```python
def aerodynamic_resistance(u_b, z_b, z0m, h_veg=None, disp=None, z0h=None, stability=None):
    """
    Computes the aerodynamic resistance for heat or vapor transport.
    
    Parameters:
    -----------
    u_b : float or array
        Wind speed at height z_b [m s^-1]
        Measured or modeled wind speed at reference height
        
    z_b : float
        Reference height for wind speed measurement [m]
        Height at which wind speed is measured or modeled
        
    z0m : float or array
        Roughness length for momentum transfer [m]
        Parameter characterizing the surface roughness for momentum
        
    h_veg : float or array, optional
        Vegetation height [m]
        Used to estimate displacement height if not provided directly
        
    disp : float or array, optional
        Displacement height [m]
        Effective height at which momentum absorption occurs
        
    z0h : float or array, optional
        Roughness length for heat transfer [m]
        Parameter characterizing the surface roughness for heat
        
    stability : float or array, optional
        Atmospheric stability parameter
        Corrects for non-neutral atmospheric conditions
        
    Returns:
    --------
    ra : float or array
        Aerodynamic resistance [s m^-1]
        Resistance against transfer between surface and atmosphere
    """
    # Set defaults if not provided
    if disp is None and h_veg is not None:
        disp = 0.67 * h_veg
    
    if z0h is None:
        z0h = 0.1 * z0m
    
    # Calculate friction velocity
    k = 0.41  # von Karman constant
    u_star = k * u_b / np.log((z_b - disp) / z0m)
    
    # Basic aerodynamic resistance
    ra = np.log((z_b - disp) / z0h) / (k * u_star)
    
    # Apply stability correction if provided
    if stability is not None:
        ra = ra * stability_correction(stability)
    
    return ra
```

## Validation and Limitations

### Validation Approaches

- Comparison with eddy covariance flux tower measurements
- Water balance validation at catchment scale
- Cross-validation with other ET products (e.g., MODIS ET)
- Energy balance closure assessment

### Known Limitations

1. **Energy Balance Closure**:
   - The energy balance often does not close perfectly in real-world measurements
   - This leads to systematic uncertainties in ET estimates

2. **Limited Temporal Resolution**:
   - Satellite revisit constraints limit the temporal frequency of estimates
   - Cloud cover further reduces data availability

3. **Parameter Uncertainty**:
   - Surface and aerodynamic resistances are difficult to parameterize accurately
   - Assumptions in downscaling may introduce errors

4. **Scale Issues**:
   - Meteorological data resolution mismatch with satellite data
   - Sub-pixel heterogeneity effects on coarser resolution inputs

## References

1. Bastiaanssen, W. G., et al. (2012). "The ETLook model: a novel approach for estimating evapotranspiration from remote sensing data." International Journal of Applied Earth Observation and Geoinformation.

2. Allen, R. G., et al. (1998). "Crop evapotranspiration: Guidelines for computing crop water requirements." FAO Irrigation and Drainage Paper 56.

3. FAO (2020). "WaPOR: The FAO portal to monitor Water Productivity through Open access of Remotely sensed derived data."

4. Su, Z. (2002). "The Surface Energy Balance System (SEBS) for estimation of turbulent heat fluxes." Hydrology and Earth System Sciences.

5. Gao, F., et al. (2012). "Toward mapping surface energy fluxes at field scales using MODIS and Landsat data." Remote Sensing of Environment.
