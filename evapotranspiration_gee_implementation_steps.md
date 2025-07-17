# Implementing PyWaPOR Actual Evapotranspiration with Google Earth Engine

## Process Overview with Difficulty Ratings

### Step 1: Setting Up Google Earth Engine (Difficulty: Low)
- Create a GEE account if you don't have one
- Set up the Python API client using `earthengine-api`
- Authenticate your account

### Step 2: Defining Your Area of Interest (Difficulty: Low)
- Import your polygon as a GEE Feature or FeatureCollection
- Set up time period parameters for multi-year analysis

### Step 3: Data Acquisition (Difficulty: Medium)
- **Optical data:** Import Sentinel-2 collection for NDVI and albedo
  - Filter by date range, cloud cover, and location
- **Thermal data:** Import Landsat 8/9 collection for LST
  - Filter by date range, cloud cover, and location
- **Meteorological data:** Import ERA5-Land collection
  - Select relevant bands (temperature, pressure, humidity, wind, radiation)
- **Precipitation data:** Import CHIRPS dataset
  - Filter by date range and location

### Step 4: Preprocessing (Difficulty: Medium)
- Apply cloud masks to optical and thermal data
- Calculate vegetation indices (NDVI, EVI)
- Calculate albedo from surface reflectance
- Convert Landsat thermal bands to land surface temperature
- Resample meteorological data to match study area
- Align all datasets to common grid and temporal resolution

### Step 5: Vegetation Parameter Calculation (Difficulty: Medium)
```javascript
// GEE implementation for vegetation cover and LAI
function vegetationCover(ndvi) {
  var ndMin = 0.125;
  var ndMax = 0.8;
  var vcPow = 0.7;
  
  return ee.Image(1).subtract(
    ee.Image(ndMax).subtract(ndvi)
    .divide(ee.Image(ndMax).subtract(ndMin))
    .pow(vcPow)
  )
  .clamp(0, 1);
}

function leafAreaIndex(vc) {
  // LAI = -ln(1-fc)/k where k is extinction coefficient
  var k = 0.5; // typical value for extinction coefficient
  return ee.Image(1).subtract(vc).log().multiply(-1).divide(k);
}

var vc = vegetationCover(ndviImage);
var lai = leafAreaIndex(vc);
```

### Step 6: Radiation Balance Calculation (Difficulty: High)
```javascript
function netRadiation(albedo, raHorClear, emissSurf, lstK, emissAtm, tAirK) {
  var stefanBoltzmann = 5.67e-8;
  
  // Net shortwave radiation
  var rNs = ee.Image(1).subtract(albedo).multiply(raHorClear);
  
  // Net longwave radiation
  var rNl = emissSurf.multiply(
    emissAtm.multiply(stefanBoltzmann).multiply(tAirK.pow(4))
    .subtract(stefanBoltzmann.multiply(lstK.pow(4)))
  );
  
  // Total net radiation
  return rNs.add(rNl);
}

// Partition radiation between soil and canopy
function radiationPartitioning(netRadiation, vc) {
  var netRadSoil = netRadiation.multiply(ee.Image(1).subtract(vc));
  var netRadCanopy = netRadiation.multiply(vc);
  return {soil: netRadSoil, canopy: netRadCanopy};
}
```

### Step 7: Soil Heat Flux Calculation (Difficulty: Medium)
```javascript
function soilHeatFlux(rn_soil, g0) {
  // g0 typically ranges from 0.2 (vegetated) to 0.35 (bare soil)
  return rn_soil.multiply(g0);
}

var g0 = ee.Image(0.35).subtract(vc.multiply(0.15)); // g0 varies with vegetation cover
var gSoil = soilHeatFlux(rnPartitioned.soil, g0);
```

### Step 8: Meteorological Parameter Processing (Difficulty: High)
- Calculate air density
- Calculate vapor pressure deficit
- Calculate atmospheric emissivity
- Calculate psychrometric constant

```javascript
// Example for air density calculation
function airDensity(tAirK, specificHumidity, pressure) {
  var dryAirConstant = 287.04; // J/kg/K
  var waterVaporConstant = 461.5; // J/kg/K
  
  var virtualTemp = tAirK.multiply(
    ee.Image(1).add(specificHumidity.multiply(waterVaporConstant.divide(dryAirConstant).subtract(1)))
  );
  
  return pressure.divide(dryAirConstant.multiply(virtualTemp));
}

var airDens = airDensity(tAirK, specificHumidity, pressure);
```

### Step 9: Resistance Calculation (Difficulty: Very High)
- Calculate aerodynamic resistances
- Calculate surface resistances

```javascript
// Aerodynamic resistance calculation - simplified version
function aerodynamicResistance(windSpeed, vegHeight) {
  var vonKarman = 0.41;
  var heightWind = 10; // Wind measurement height (m)
  var z0m = vegHeight.multiply(0.123); // Roughness length for momentum
  var displacement = vegHeight.multiply(0.67); // Displacement height
  
  // Friction velocity
  var uStar = vonKarman.multiply(windSpeed).divide(
    ee.Image(heightWind).subtract(displacement).divide(z0m).log()
  );
  
  // Resistance (simplified, neutral conditions)
  var z0h = z0m.multiply(0.1); // Roughness length for heat
  return ee.Image(heightWind).subtract(displacement).divide(z0h).log().divide(
    vonKarman.multiply(uStar)
  );
}

// Surface resistance for soil - depends on soil moisture
function soilResistance(soilMoisture) {
  var soilResFactor = 3500; // Resistance factor
  return soilResFactor.multiply(
    ee.Image(1).subtract(soilMoisture).divide(soilMoisture)
  );
}

// Canopy resistance - simplified Jarvis approach
function canopyResistance(lai, solarRad, vpd, tAir, soilMoisture) {
  var rsMin = 100; // Minimum stomatal resistance (s/m)
  
  // Environmental stress factors
  var fSolar = ee.Image(1).subtract(ee.Image(-0.5).multiply(solarRad).exp());
  var fVpd = ee.Image(1).divide(ee.Image(1).add(vpd.multiply(0.5)));
  var fTemp = ee.Image(1).divide(
    ee.Image(1).add(ee.Image(-0.2).multiply(tAir.subtract(298)).pow(2))
  );
  var fSoil = soilMoisture.divide(
    soilMoisture.add(ee.Image(0.3))
  );
  
  // Effective LAI for transpiration
  var laiActive = lai.multiply(1.5).clamp(0, 3);
  
  // Canopy resistance (rs / LAI_active)
  return rsMin.divide(laiActive).multiply(
    fSolar.multiply(fVpd).multiply(fTemp).multiply(fSoil).pow(-1)
  );
}

var rAero = aerodynamicResistance(windSpeed, vegHeight);
var rSoil = soilResistance(soilMoisture);
var rCanopy = canopyResistance(lai, solarRad, vpdAir, tAir, rootZoneMoisture);
```

### Step 10: Evaporation and Transpiration Calculation (Difficulty: Very High)
- Calculate soil evaporation using Penman-Monteith
- Calculate plant transpiration using Penman-Monteith
- Calculate interception evaporation

```javascript
// Generic Penman-Monteith equation
function penmanMonteith(rn, g, airDens, vpSat, vpAct, psychro, deltaVp, ra, rs) {
  var cp = 1013; // Specific heat of air (J/kg/K)
  
  var numerator = deltaVp.multiply(rn.subtract(g)).add(
    airDens.multiply(cp).multiply(vpSat.subtract(vpAct)).divide(ra)
  );
  
  var denominator = deltaVp.add(
    psychro.multiply(ee.Image(1).add(rs.divide(ra)))
  );
  
  return numerator.divide(denominator);
}

// Soil evaporation
var leSoil = penmanMonteith(
  rnPartitioned.soil, gSoil, airDens, 
  vpSatSoil, vpAir, psychro, deltaSvpSoil, 
  rAeroSoil, rSoil
);

// Transpiration
var leTransp = penmanMonteith(
  rnPartitioned.canopy, ee.Image(0), airDens,
  vpSatLeaf, vpAir, psychro, deltaSvpLeaf,
  rAeroCanopy, rCanopy
);

// Interception evaporation (simplified)
var precipIntercepted = precipitation.multiply(
  ee.Image(1).subtract(ee.Image(-0.5).multiply(lai).exp())
).clamp(0, ee.Image(0.2).multiply(lai));

var leInterception = precipIntercepted.multiply(2.45e6).divide(86400);
```

### Step 11: Total Evapotranspiration Calculation (Difficulty: Low)
```javascript
function totalEvapotranspiration(leSoil, leTransp, leIntercept) {
  // Sum latent heat components
  var leTotal = leSoil.add(leTransp).add(leIntercept);
  
  // Convert from W/m² to mm/day
  // 1 W/m² = 0.0864 mm/day
  return leTotal.multiply(0.0864);
}

var actualET = totalEvapotranspiration(leSoil, leTransp, leInterception);
```

### Step 12: Downscaling (Difficulty: Very High)
- Use machine learning approach to downscale ET to higher resolution
- Train a model using high-resolution optical bands and lower-resolution ET

```javascript
// This is a complex process requiring machine learning implementation
// Below is a conceptual example

// Collect training data
var trainingFeatures = ee.Image.cat([
  ndvi, albedo, lst.subtract(tAir),
  elevation, slope, aspect
]);

// Train a random forest model
var trainedModel = ee.Classifier.smileRandomForest(100)
  .train({
    features: trainingPoints,
    classProperty: 'et_value',
    inputProperties: trainingFeatures.bandNames()
  });

// Apply the model to create high-resolution ET
var etHighRes = trainingFeatures.classify(trainedModel);
```

### Step 13: Validation and Quality Control (Difficulty: Medium)
- Compare with available ground measurements
- Analyze time series consistency
- Check for artifacts and anomalies
- Apply quality filters to final product

### Step 14: Exporting Results (Difficulty: Low)
```javascript
Export.image.toDrive({
  image: actualET,
  description: 'actual_evapotranspiration',
  scale: 10,
  region: areaOfInterest,
  fileFormat: 'GeoTIFF'
});
```

## Overall Complexity Assessment

| Step | Complexity | Time Investment | Expertise Required |
|------|------------|-----------------|-------------------|
| 1-2 | Low | 1-2 hours | Basic GEE knowledge |
| 3-4 | Medium | 8-16 hours | Intermediate remote sensing |
| 5 | Medium | 4-8 hours | Basic vegetation modeling |
| 6-7 | High | 16-24 hours | Energy balance understanding |
| 8 | High | 16-24 hours | Meteorological expertise |
| 9 | Very High | 24-40 hours | Advanced micrometeorology |
| 10 | Very High | 24-40 hours | Evapotranspiration modeling expertise |
| 11 | Low | 1-2 hours | Basic calculations |
| 12 | Very High | 40+ hours | Advanced machine learning |
| 13-14 | Medium | 8-16 hours | Data analysis skills |

## Alternative Approaches to Reduce Complexity

1. **Use simplified resistance formulations** instead of full Monin-Obukhov implementations

2. **Adopt empirical relationships** for ET components based on vegetation indices and LST

3. **Skip downscaling** and work with coarser resolution outputs

4. **Use existing GEE ET products** as baselines and apply regional corrections

5. **Focus on clear-sky days only** to avoid complex atmospheric corrections

## Comparison with Soil Moisture Calculation

Both ET and soil moisture calculations share several components:

1. **Common inputs**: Both require LST, NDVI, and meteorological data
2. **Similar complexity areas**: Resistance calculations and downscaling
3. **Main difference**: ET calculation has more components (transpiration, interception) and requires additional parameters

The ET implementation is generally more complex than soil moisture, primarily due to:
- More energy balance components
- Multiple resistances for different pathways
- More complex downscaling due to ET's higher spatial variability
