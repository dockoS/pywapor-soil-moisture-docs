# PyWaPOR Technical Documentation

This repository contains technical documentation for understanding and customizing both the soil moisture and evapotranspiration calculations in the PyWaPOR library.

## Contents

### Evapotranspiration Documentation

1. **[PyWaPOR ET Exact Constants](pywapor_et_exact_constants.md)** - Complete collection of all exact constants, parameters, and formulas extracted directly from the PyWaPOR source code for evapotranspiration calculations.

2. **[ET GEE Data Sources](et_gee_data_sources.md)** - Comprehensive mapping of each parameter required for the ET calculation to its corresponding Google Earth Engine data source, including processing guidance and code examples.

3. **[PyWaPOR ET Calculation Workflow](pywapor_et_calculation_workflow.md)** - Step-by-step guide detailing the complete workflow for implementing PyWaPOR's evapotranspiration calculation, showing how all parameters interact in the calculation chain.

### Soil Moisture Documentation

1. **[Relative Soil Moisture Technical Documentation](relative_soil_moisture_technical_documentation.md)** - Detailed explanation of the theoretical framework, implementation, and code details for relative soil moisture calculation in PyWaPOR.

2. **[Soil Moisture Parameter Sources](soil_moisture_parameter_sources.md)** - Comprehensive guide on where to obtain each parameter required for the soil moisture calculation, including data sources and preprocessing steps.

3. **[Soil Moisture GEE Implementation Steps](soil_moisture_gee_implementation_steps.md)** - Step-by-step guide for implementing PyWaPOR's soil moisture calculation in Google Earth Engine.

## Purpose

This documentation is intended for researchers and developers who want to:
- Understand the scientific principles behind PyWaPOR's soil moisture and evapotranspiration calculations
- Customize these calculations for specific applications
- Locate and prepare all necessary input data for the calculations
- Implement these calculations in Google Earth Engine for high-resolution results
- Achieve accurate, 10-meter resolution outputs consistent with PyWaPOR standards

## Key Features Documented

### Soil Moisture Features
1. The Triangle Method for soil moisture estimation
2. NDVI-based vegetation cover calculation
3. Temperature extremes calculation for wet and dry conditions
4. Decision tree-based thermal sharpening for high resolution
5. Data sources and preprocessing workflow

### Evapotranspiration Features
1. Energy balance approach based on the ETLook algorithm
2. Penman-Monteith equations for evaporation and transpiration components
3. Plant stress factors (radiation, temperature, moisture, VPD) calculations
4. Resistance formulations for canopy and soil
5. Google Earth Engine implementation workflow for all components

## Practical Applications

### Crop Water Requirement Calculation

One of the primary applications of accurate evapotranspiration data is calculating crop water requirements. After obtaining the evapotranspiration value, the crop water requirement (Qm) can be calculated using the following formula:

```
Qm = (Kc × ETP - Peff) / [Eff × (1 - Lf)]
```

Where:
- **Qm** = Water requirement (mm or m³/ha)
- **Kc** = Crop coefficient (specific to each crop and growth stage)
- **ETP** = Evapotranspiration (mm)
- **Peff** = Effective precipitation (mm)
- **Eff** = Irrigation system efficiency (as a decimal, e.g., 0.75 for 75% efficiency)
- **Lf** = Leaching fraction (as a decimal, 0 for non-saline soils)

Example calculation for tomatoes during peak season:
- Kc for tomatoes (peak) = 1.1
- Monthly ETP = 213.28 mm
- Peff = 0 mm (negligible rainfall)
- Eff = 0.75 (sprinkler irrigation)
- Lf = 0 (non-saline soil)

```
Qm = (1.1 × 213.28 - 0) / [0.75 × (1 - 0)] = 312.8 mm = 3,128 m³/ha
```

This calculation provides the net irrigation water requirement, essential for efficient irrigation scheduling and water resource management.
