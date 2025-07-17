# PyWaPOR Evapotranspiration: Exact Constants and Formulations

This document provides the exact constants and mathematical formulations used in the PyWaPOR library for calculating actual evapotranspiration. These values are extracted directly from the PyWaPOR source code to ensure precise implementation.

## Table of Contents

1. [Physical Constants](#physical-constants)
2. [Vegetation Parameters](#vegetation-parameters)
3. [Resistance Parameters](#resistance-parameters)
4. [Surface Parameters](#surface-parameters)
5. [Meteorological Parameters](#meteorological-parameters)
6. [Stress Formulations](#stress-formulations)
7. [Roughness Formulations](#roughness-formulations)
8. [Evapotranspiration Components](#evapotranspiration-components)
9. [Conversion Formulations](#conversion-formulations)

## Physical Constants

These are the fundamental physical constants used throughout the PyWaPOR calculations:

| Constant | Value | Description | Units |
|----------|-------|-------------|-------|
| `zero_celcius` | 273.15 | 0 degrees C in Kelvin | K |
| `g` | 9.807 | Gravitational acceleration | m s⁻² |
| `gc_spec` | 287.0 | Gas constant for dry air | J kg⁻¹ K⁻¹ |
| `gc_dry` | 2.87 | Dry air gas constant | mbar K⁻¹ m³ kg⁻¹ |
| `gc_moist` | 4.61 | Moist air gas constant | mbar K⁻¹ m³ kg⁻¹ |
| `r_mw` | 0.622 | Ratio of molecular weights water/air | - |
| `sh` | 1004.0 | Specific heat of air | J kg⁻¹ K⁻¹ |
| `lh_0` | 2501000.0 | Latent heat of evaporation at 0°C | J kg⁻¹ |
| `lh_rate` | -2361 | Rate of latent heat vs temperature | J kg⁻¹ °C⁻¹ |
| `k` | 0.41 | von Karman constant | - |
| `sb` | 5.67e-8 | Stefan-Boltzmann constant | W m⁻² K⁻⁴ |
| `day_sec` | 86400.0 | Seconds in a day | s |

## Vegetation Parameters

These parameters are used for vegetation-related calculations:

| Parameter | Value | Description | Units |
|-----------|-------|-------------|-------|
| `nd_min` | 0.125 | NDVI minimum for vegetation cover calculation | - |
| `nd_max` | 0.8 | NDVI maximum for vegetation cover calculation | - |
| `vc_pow` | 0.7 | Power parameter for vegetation cover formula | - |
| `vc_min` | 0 | Minimum vegetation cover | - |
| `vc_max` | 0.9677324224821418 | Maximum vegetation cover | - |
| `lai_pow` | -0.45 | Power parameter for LAI calculation | - |

### Vegetation Cover Formula

```
vc = 1 - ((nd_max - ndvi) / (nd_max - nd_min)) ** vc_pow
vc = np.clip(vc, vc_min, vc_max)
```

## Resistance Parameters

### Soil Resistance

| Parameter | Value | Description | Units |
|-----------|-------|-------------|-------|
| `r_soil_min` | 800 | Minimum soil resistance | s m⁻¹ |
| `r_soil_pow` | -2.1 | Power in soil resistance function | - |
| `z0_soil` | 0.001 | Soil roughness length | m |

### Soil Resistance Formula

```python
r_soil = r_soil_min * se_top ** r_soil_pow
```

Where:
- `se_top` is the top soil effective saturation [-]

### Canopy Resistance

| Parameter | Value | Description | Units |
|-----------|-------|-------------|-------|
| `rs_min` | 70 | Minimum stomatal resistance | s m⁻¹ |
| `rcan_max` | 1000000.0 | Maximum canopy resistance | s m⁻¹ |

### Canopy Resistance Formulas

```python
# Atmospheric canopy resistance (without soil moisture stress)
r_canopy_0 = (rs_min / lai_eff) / (stress_rad * stress_temp * stress_vpd)

# Full canopy resistance (with soil moisture stress)
r_canopy = r_canopy_0 / stress_moist
```

## Surface Parameters

### Land Surface Parameters

| Parameter | Value | Description | Units |
|-----------|-------|-------------|-------|
| `emiss_bare` | 0.95 | Emissivity of bare soil | - |
| `emiss_full` | 0.99 | Emissivity of full vegetation | - |
| `r0_bare` | 0.38 | Albedo of dry bare soil | - |

## Meteorological Parameters

| Parameter | Value | Description | Units |
|-----------|-------|-------------|-------|
| `lapse` | -0.0065 | Temperature lapse rate | K m⁻¹ |
| `power` | (g/(-lapse*gc_spec)) | Exponent for pressure-height relation | - |

## Stress Formulations

### Radiation Stress

```python
stress_rad = ra_24/(ra_24 + 60.) * (1 + 60./500.)
stress_rad = np.clip(stress_rad, 0, 1)
```

Where:
- `ra_24` is the daily solar radiation [W m⁻²]

### Moisture Stress

```python
stress_moist = tenacity * se_root - (np.sin(2*np.pi*se_root)) / (2*np.pi)
stress_moist = np.clip(stress_moist, 0, 1)
```

Where:
- `se_root` is the root zone effective saturation [-]
- `tenacity` is typically 1.5 for moderately sensitive plants [-]

### Temperature Stress

```python
f = (t_max - t_opt) / (t_opt - t_min)
x = (t_air_24 - t_min) * (t_max - t_air_24)**f
y = (t_opt - t_min) * (t_max - t_opt)**f
stress_temp = x/y
stress_temp = np.clip(stress_temp, 0, 1)
```

Where:
- `t_air_24` is the daily air temperature [°C]
- `t_opt` = 25.0°C (optimal temperature for plant growth)
- `t_min` = 0.0°C (minimum temperature for plant growth)
- `t_max` = 50.0°C (maximum temperature for plant growth)

### Vapor Pressure Deficit Stress

```python
stress_vpd = vpd_slope * np.log(0.1 * vpd_24 + 0.5) + 1
stress_vpd = np.clip(stress_vpd, 0, 1)
```

Where:
- `vpd_24` is the daily vapor pressure deficit [mbar]
- `vpd_slope` is typically -0.3 to -0.7 [mbar⁻¹]

## Roughness Formulations

### Obstacle Height

```python
def obstacle_height(ndvi, z_obst_max, ndvi_obs_min=0.25, ndvi_obs_max=0.75, obs_fr=0.25):
    if ndvi <= ndvi_obs_min:
        return obs_fr * z_obst_max
    elif ndvi >= ndvi_obs_max:
        return z_obst_max
    else:
        frac = obs_fr + (1-obs_fr) * (ndvi-ndvi_obs_min) / (ndvi_obs_max-ndvi_obs_min)
        return frac * z_obst_max
```

Where:
- `ndvi` is the normalized difference vegetation index [-]
- `z_obst_max` is the maximum obstacle height [m]
- `ndvi_obs_min` = 0.25 (NDVI at minimum obstacle height)
- `ndvi_obs_max` = 0.75 (NDVI at maximum obstacle height)
- `obs_fr` = 0.25 (ratio of min to max obstacle height)

### Displacement Height

```python
def displacement_height(lai, z_obst, c1=12):
    if lai == 0:
        return 0
    else:
        return z_obst * (1 - (1 - np.exp(-np.sqrt(c1 * lai))) / (np.sqrt(c1 * lai)))
```

Where:
- `lai` is the leaf area index [-]
- `z_obst` is the obstacle height [m]
- `c1` = 12 (coefficient for LAI-based displacement height)

### Roughness Length (for vegetation)

```python
def veg_roughness(z_obst, disp, z_obst_max, lai, z_oro):
    veg = 0.193
    z_dif = z_obst - disp
    
    term1 = np.minimum(0.41**2/((np.log(z_dif/(0.002*z_obst_max))+veg)**2), 1.0) + 0.35*lai/2
    term2 = np.exp(0.41/np.minimum(np.sqrt(term1), 0.3)-veg)
    
    return z_dif/term2 + z_oro
```

Where:
- `z_obst` is the obstacle height [m]
- `disp` is the displacement height [m]
- `z_obst_max` is the maximum obstacle height [m]
- `lai` is the leaf area index [-]
- `z_oro` is the orographic roughness [m]

## Evapotranspiration Components

### Interception Formula

```python
def interception_mm(P_24, vc, lai, int_max=0.2):
    if vc == 0 or lai == 0 or P_24 == 0:
        return 0
    else:
        return int_max * lai * (1 - (1 / (1 + ((vc * P_24) / (int_max * lai)))))
```

Where:
- `P_24` is the daily rainfall [mm day⁻¹]
- `vc` is the vegetation cover [-]
- `lai` is the leaf area index [-]
- `int_max` = 0.2 (maximum interception per leaf) [mm day⁻¹]

### Reference Evapotranspiration

```python
def et_reference(rn_24_grass, ad_24, psy_24, vpd_24, ssvp_24, u_24):
    r_grass = 70  # standard grass resistance [s m⁻¹]
    ra_grass = 208. / u_24  # aerodynamic resistance for grass [s m⁻¹]
    
    et_ref_24 = (ssvp_24 * rn_24_grass + ad_24 * sh * (vpd_24 / ra_grass)) / \
        (ssvp_24 + psy_24 * (1 + r_grass / ra_grass))
        
    return et_ref_24
```

Where:
- `rn_24_grass` is the net radiation for reference grass surface [W m⁻²]
- `ad_24` is the daily air density [kg m⁻³]
- `psy_24` is the daily psychrometric constant [mbar K⁻¹]
- `vpd_24` is the daily vapor pressure deficit [mbar]
- `ssvp_24` is the daily slope of saturated vapor pressure curve [mbar K⁻¹]
- `u_24` is the daily wind speed at observation height [m s⁻¹]

## Conversion Formulations

### Energy to Water Depth Conversion

```python
et_ref_24_mm = et_ref_24 * day_sec / lh_24
```

Where:
- `et_ref_24` is the daily reference evapotranspiration energy equivalent [W m⁻²]
- `day_sec` = 86400.0 (seconds in a day) [s]
- `lh_24` is the daily latent heat of evaporation [J kg⁻¹]

### Total Actual Evapotranspiration

```python
def eti_actual_mm(e_24_mm, t_24_mm, int_mm):
    return e_24_mm + t_24_mm + int_mm
```

Where:
- `e_24_mm` is the daily evaporation [mm day⁻¹]
- `t_24_mm` is the daily transpiration [mm day⁻¹]
- `int_mm` is the daily interception [mm day⁻¹]
