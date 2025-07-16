## Algorithm Specification: Converting Relative Soil Moisture (RSM) to Absolute Soil Water Content using ISRIC SoilGrids

### 1. Purpose
Provide an implementation‑ready workflow that transforms a **Relative Soil Moisture (RSM)** index (0–1) into **absolute volumetric water content (θₜ, m³ m⁻³) and water storage (Wₜ, mm m⁻¹)**, using only ISRIC SoilGrids v2.1 layers:

* **Field capacity** (θ_fc : volumetric water content at −33 kPa)
* **Wilting point** (θ_wp : volumetric water content at −1500 kPa)

### 2. Input Data Requirements
| Symbol | SoilGrids API layer | Depths (cm)* | Native unit | Notes |
|--------|--------------------|--------------|-------------|-------|
| θ_fc   | `WCFC`             | 0–5, 5–15, 15–30, 30–60, 60–100, 100–200 | 10⁻³ cm³ cm⁻³ (each integer = 0.001 m³ m⁻³) | Field capacity (−33 kPa) |
| θ_wp   | `WCWP`             | same         | same        | Wilting point (−1500 kPa) |
| RSMₜ   | user‑supplied      | root‑zone    | –           | 0 → θ_wp ; 1 → θ_fc |

*SoilGrids provides each depth as a band; match depths before averaging.*

### 3. Pre‑processing
1. **Clip & resample** SoilGrids bands to the AOI at native 250 m resolution.
2. **Unit conversion:** divide raw integers by 1000 to obtain fractions (m³ m⁻³).
3. **Root‑zone averaging:** depth‑weight θ_fc and θ_wp across the same root‑zone thickness used by the RSM product (e.g., 0–100 cm).

### 4. Core Algorithm
#### 4.1 RSM definition
* 0 = θ_wp (dry limit)  
* 1 = θ_fc (field capacity)  

RSM therefore measures the fraction of plant‑available water that has been refilled.

#### 4.2 Compute volumetric water content
θₜ = θ_wp + RSMₜ · (θ_fc − θ_wp)

#### 4.3 Convert to water storage
For a root‑zone depth Δz (m): Wₜ = θₜ × Δz × 1000 mm m⁻¹

*Example:* Δz = 1 m, θ_wp = 0.10, θ_fc = 0.30, RSMₜ = 0.4 ⇒ θₜ = 0.18, Wₜ = 180 mm m⁻¹.

### 5. Implementation Notes
* **Clipping:** restrict RSM values to 0–1.
* **Temporal alignment:** ensure RSM timestamp matches ancillary stack.
* **Static soil map uncertainty:** SoilGrids values are climatological medians; expect ±0.05 m³ m⁻³ local error.
* **Validation:** compare derived θₜ with in‑situ probes; compute RMSE and bias.

### 6. References
1. ISRIC – World Soil Information (2020): *SoilGrids 250 m v2.1 Documentation*.
2. Hillel, D. (1998): *Environmental Soil Physics*. Academic Press.
3. Tóth, B. et al. (2017): “Mapping field capacity and wilting point across Europe …” *Hydrol. Earth Syst. Sci.* 21, 1989‑2011.
