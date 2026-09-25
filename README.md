# marine-eye
rajmohan-05218/marine-eye
# Marine Oil Spill Detection and Vessel Attribution System

An AI- and GIS-based marine intelligence platform that detects probable oil slicks from satellite imagery, models their ocean drift, and ranks potential source vessels using historic AIS data and scenario-based spill simulation.

> **Core idea:**  
> The system does not only ask, “Which vessels were near the spill?”  
> It asks, “If each candidate vessel had released oil at its recorded AIS location and time, would the simulated slick reach and match the oil slick observed in satellite imagery?”

---

## Problem Statement

Marine oil spills cause severe damage to marine ecosystems, fisheries, coastal communities, and protected habitats. In many cases, the vessel responsible for an illegal discharge cannot be easily identified because:

- Oil slicks drift after release due to ocean currents, wind, waves, and diffusion.
- A satellite image records the slick only at the time of satellite observation, not necessarily at the time or place of release.
- Many vessels may be present near the observed slick.
- AIS data can contain gaps, delays, or incomplete vessel information.
- Dark regions in SAR imagery may be oil slicks or natural look-alikes such as low-wind areas, algae, internal waves, or biogenic films.

This project provides an automated investigation-support workflow to:

1. Detect and segment possible oil slicks from SAR satellite imagery.
2. Calculate spill geometry such as area, perimeter, centroid, orientation, and coastal proximity.
3. Estimate the slick’s probable origin and release-time window through backward drift modelling.
4. Predict the future movement of the slick using forward drift simulation.
5. Analyse historic AIS vessel traffic around the probable source region.
6. Simulate “what-if” spill scenarios for candidate vessels.
7. Rank vessels based on explainable spatio-temporal and drift-match evidence.
8. Present results through an interactive GIS dashboard.

---

## Objectives

- Detect probable marine oil slicks from Sentinel-1 SAR imagery.
- Differentiate oil candidates from possible SAR look-alikes using AI confidence and rule-based validation.
- Extract oil-spill geometry and spatial characteristics.
- Perform backward trajectory modelling to estimate probable release region and release time.
- Perform forward trajectory modelling to estimate likely future spill movement.
- Reconstruct nearby vessel routes from historic AIS data.
- Filter irrelevant vessel traffic using location, time, vessel type, and route constraints.
- Execute scenario-based forward simulations for each candidate vessel.
- Generate an explainable vessel-confidence score.
- Support authorities with a map-based incident investigation interface.

---

## Key Innovation

### Scenario-Based Vessel Attribution

Most basic systems rank vessels only by their distance from an observed spill.

This system uses a stronger evidence-based approach:

```text
For each candidate vessel:

1. Select its AIS position at a possible release time.
2. Assume oil was released at that point.
3. Run a forward oil-drift simulation using wind and ocean currents.
4. Predict the slick location at the satellite observation time.
5. Compare the simulated slick with the actual satellite-detected slick.
6. Assign a drift-match score.
7. Combine it with location, timing, route, and AIS-behaviour evidence.
```

```text
Question tested by the system:

“If Vessel A released oil at this location and time,
would the oil slick be near the currently observed spill?”
```

The candidate vessel whose simulated oil distribution most closely matches the observed satellite slick receives a higher investigation confidence score.

> The output is a **ranked list of potential source vessels for human investigation**, not an automatic legal accusation.

---

## System Architecture

```text
┌──────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                               │
│                                                                      │
│ Sentinel-1 SAR │ Sentinel-2 EO │ Historic AIS │ Wind │ Currents     │
│ Coastline GIS  │ Protected Areas │ Ports │ Shipping Lanes            │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     DATA INGESTION AND STORAGE                       │
│                                                                      │
│ Satellite Downloader │ AIS Ingestion │ Ocean/Weather Data Ingestion  │
│ Raw File Storage │ PostgreSQL + PostGIS                              │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  SATELLITE PROCESSING AND AI LAYER                  │
│                                                                      │
│ SAR Calibration → Noise Removal → Speckle Filtering → Land Masking  │
│                     ↓                                                │
│               U-Net / DeepLabV3+ Segmentation                       │
│                     ↓                                                │
│        Oil-Slick Mask + Confidence + Geographic Polygon             │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      SPILL ANALYSIS LAYER                           │
│                                                                      │
│ Area │ Perimeter │ Centroid │ Length │ Width │ Orientation           │
│ Coast Distance │ Protected-Area Intersection │ Look-Alike Risk       │
└───────────────┬──────────────────────────────────────┬───────────────┘
                │                                      │
                ▼                                      ▼
┌───────────────────────────────┐       ┌──────────────────────────────┐
│ OIL-DRIFT MODELLING ENGINE    │       │ AIS VESSEL ANALYTICS ENGINE  │
│                               │       │                              │
│ Backward Hindcast             │       │ Clean AIS messages           │
│ Probable Origin Region        │       │ Reconstruct vessel tracks    │
│ Release-Time Window           │       │ Detect AIS gaps              │
│ Forward Spill Forecast        │       │ Detect route/speed anomalies │
└───────────────┬───────────────┘       └──────────────┬───────────────┘
                │                                      │
                └──────────────────┬───────────────────┘
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│              SCENARIO-BASED VESSEL ATTRIBUTION ENGINE                │
│                                                                      │
│ Candidate Vessel AIS Position + Release Time                         │
│                     ↓                                                │
│ Forward Oil-Drift Simulation                                         │
│                     ↓                                                │
│ Compare Simulated Slick vs Observed Satellite Slick                  │
│                     ↓                                                │
│ Explainable Candidate Vessel Ranking                                 │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        DASHBOARD AND REPORTING                       │
│                                                                      │
│ Spill Map │ Drift Map │ AIS Tracks │ Candidate Ranking │ Evidence    │
│ Incident Report │ GeoJSON/CSV Export │ Investigation Timeline        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## System Workflow

```text
1. Acquire Sentinel-1 SAR image for selected marine region
                    ↓
2. Preprocess image and remove noise/land regions
                    ↓
3. Run AI segmentation model to detect possible oil slick
                    ↓
4. Convert detected mask into geographic spill polygon
                    ↓
5. Calculate spill area, geometry, location, and confidence
                    ↓
6. Run backward drift model to estimate probable source region
                    ↓
7. Retrieve historic AIS vessel tracks for source time window
                    ↓
8. Filter irrelevant vessels by distance, time, and vessel type
                    ↓
9. Run “what-if” forward spill simulation for each candidate vessel
                    ↓
10. Compare simulated slick with observed satellite slick
                    ↓
11. Rank vessels using explainable evidence score
                    ↓
12. Display results in the GIS investigation dashboard
```

---

## Main Modules

### 1. Satellite Data Processing Module

This module processes Sentinel-1 SAR imagery before AI inference.

#### Processing steps

```text
Raw Sentinel-1 GRD Image
        ↓
Orbit File Correction
        ↓
Thermal Noise Removal
        ↓
Radiometric Calibration
        ↓
Speckle Noise Filtering
        ↓
Terrain / Geometric Correction
        ↓
Convert Backscatter to dB
        ↓
Land and Coastline Masking
        ↓
Image Tiling for AI Model
```

#### Output

- Processed SAR raster image
- VV and VH backscatter layers
- Georeferenced image tiles
- Satellite acquisition timestamp
- Geographic bounds of the image

---

### 2. Oil Slick Detection Module

The system uses semantic segmentation to identify the exact boundary of a possible oil slick.

#### Input

- Sentinel-1 VV polarization channel
- Sentinel-1 VH polarization channel
- VV/VH ratio
- Local texture features
- Incidence-angle information
- Optional wind-speed layer

#### Model

```text
Primary model: U-Net
Advanced model: DeepLabV3+ / Attention U-Net
Framework: PyTorch
```

#### Output

```json
{
  "slick_id": "SLICK_001",
  "satellite_time_utc": "YYYY-MM-DDTHH:MM:SSZ",
  "detection_confidence": 0.89,
  "lookalike_risk": "medium",
  "spill_polygon": "GeoJSON polygon",
  "area_km2": 8.6,
  "perimeter_km": 18.2,
  "centroid": {
    "latitude": 0.0,
    "longitude": 0.0
  },
  "orientation_degrees": 63
}
```

#### Validation checks

- Remove small isolated objects below a minimum spill-area threshold.
- Mask land and port infrastructure.
- Reject low-confidence detections.
- Identify possible low-wind or biogenic-film look-alikes.
- Compare with optical imagery where available.
- Mark uncertain detections for human review.

---

### 3. Spill Characterisation Module

The detected AI mask is transformed into a GIS polygon.

#### Calculated spill properties

| Property | Description |
|---|---|
| Spill area | Estimated surface area covered by the detected slick |
| Perimeter | Boundary length of the slick |
| Centroid | Geographic centre of the spill |
| Length and width | Dimensions derived from minimum rotated rectangle |
| Orientation | Dominant direction of the slick |
| Shape complexity | Measures fragmentation or irregularity |
| Coastal distance | Distance to nearest coastline |
| Protected-area overlap | Whether the slick intersects sensitive marine zones |
| Detection confidence | AI confidence after post-processing checks |

---

### 4. Drift Modelling Module

This module estimates the movement of oil based on ocean currents, wind, and diffusion.

#### Simplified particle-drift model

```text
Observed Slick Polygon
        ↓
Seed 100–1,000 virtual oil particles
        ↓
Use current + wind + diffusion at each time step
        ↓
Move particles backward or forward in time
        ↓
Generate probability surface / predicted slick area
```

The basic particle movement is represented as:

```text
New Position =
Current Position
+ Ocean Current Movement
+ Wind-Driven Movement
+ Diffusion / Turbulence Movement
```

#### Backward drift: hindcasting

Backward modelling estimates:

- Probable release location
- Probable release corridor
- Possible release-time window
- Uncertainty region

```text
Observed slick at satellite time
        ↓
Move particles backward 6 / 12 / 24 / 48 hours
        ↓
Generate probable origin probability map
```

#### Forward drift: forecasting

Forward modelling estimates:

- Likely future oil movement
- Coastal-impact risk
- Possible impact on ports, fishing zones, and protected areas
- Predicted spill region after 6 / 12 / 24 / 48 hours

---

### 5. AIS Vessel Analytics Module

This module processes historic AIS messages and reconstructs vessel movement.

#### AIS fields used

```text
MMSI
IMO Number
Vessel Name
Vessel Type
Timestamp (UTC)
Latitude
Longitude
Speed Over Ground (SOG)
Course Over Ground (COG)
Heading
Navigational Status
Destination
Draft
```

#### AIS processing workflow

```text
Raw AIS Messages
        ↓
Normalize timestamps to UTC
        ↓
Group records by vessel MMSI
        ↓
Sort by timestamp
        ↓
Remove duplicates and impossible positions
        ↓
Remove unrealistic speed jumps
        ↓
Interpolate limited missing positions
        ↓
Create vessel trajectory lines
        ↓
Calculate behaviour features
```

#### Vessel behaviour features

- Minimum distance to probable origin region
- Time spent inside origin-region buffer
- Vessel entry and exit time
- Vessel speed changes
- Course changes
- Route deviation
- Loitering duration
- AIS transmission gaps
- Alignment between vessel route and slick direction
- Vessel type and relevance

---

### 6. Scenario-Based Simulation Module

This is the core attribution module.

For every shortlisted vessel, the system simulates one or more possible spill-release scenarios.

#### Scenario inputs

```text
Candidate vessel MMSI
Candidate vessel AIS position
Candidate release time
Wind data for simulation period
Ocean-current data for simulation period
Satellite observation time
Observed oil-slick polygon
```

#### Scenario process

```text
Candidate Vessel Position at Time T
        ↓
Assume oil release at this point
        ↓
Seed virtual oil particles
        ↓
Run forward drift simulation until satellite observation time
        ↓
Generate simulated slick polygon
        ↓
Compare simulated slick with observed slick
        ↓
Calculate scenario drift-match score
```

#### Match calculations

The main spatial match uses Intersection over Union:

```text
Drift Match =
Area(Simulated Slick ∩ Observed Slick)
---------------------------------------
Area(Simulated Slick ∪ Observed Slick)
```

Additional comparison metrics:

- Distance between simulated and observed centroids
- Percentage of simulated particles inside observed slick
- Orientation difference
- Area similarity
- Time compatibility
- Route alignment

#### Example scenario result

```json
{
  "vessel_mmsi": "123456789",
  "scenario_release_time_utc": "YYYY-MM-DDTHH:MM:SSZ",
  "scenario_release_position": {
    "latitude": 0.0,
    "longitude": 0.0
  },
  "simulated_slick_area_km2": 7.9,
  "observed_slick_area_km2": 8.6,
  "overlap_iou": 0.71,
  "centroid_error_km": 1.8,
  "orientation_difference_degrees": 12,
  "drift_match_score": 0.82
}
```

---

### 7. Vessel Ranking Module

The system ranks vessel candidates using multiple evidence signals.

#### Final ranking formula

```text
Final Score =
0.35 × Scenario Drift Match
+ 0.20 × Proximity Score
+ 0.20 × Time Match Score
+ 0.15 × Route Alignment Score
+ 0.10 × Behaviour Anomaly Score
```

All feature scores are normalized between `0` and `1`.

#### Ranking criteria

| Criterion | Meaning |
|---|---|
| Scenario drift match | How closely the vessel’s simulated spill matches the observed slick |
| Proximity | How close the vessel was to the probable source corridor |
| Time match | Whether the vessel was present during the estimated release period |
| Route alignment | Whether vessel path is consistent with slick orientation and origin direction |
| Behaviour anomaly | Speed reduction, loitering, unexpected turn, or route deviation |
| AIS gap | Missing AIS transmissions near the probable source area |
| Vessel type | Whether vessel category is relevant for further investigation |

#### Candidate output

| Rank | Vessel | Drift Match | Proximity | Time Match | Anomaly | Final Score | Status |
|---:|---|---:|---:|---:|---:|---:|---|
| 1 | Vessel A | 0.82 | 0.91 | 0.85 | 0.62 | 0.83 | High-priority candidate |
| 2 | Vessel B | 0.48 | 0.72 | 0.63 | 0.20 | 0.56 | Medium-priority candidate |
| 3 | Vessel C | 0.17 | 0.34 | 0.41 | 0.08 | 0.26 | Low-priority candidate |

> A high score means the vessel should receive higher investigation priority. It does not independently prove legal responsibility.

---

## Technology Stack

### Data and remote sensing

- Sentinel-1 SAR imagery
- Sentinel-2 optical imagery, optional
- Historic AIS vessel data
- Wind-data sources
- Ocean-current data
- Coastline, marine-protected-area, and port GIS layers

### AI and image processing

- Python
- PyTorch
- U-Net
- DeepLabV3+
- ESA SNAP
- GDAL
- Rasterio
- OpenCV
- scikit-image
- NumPy
- scikit-learn

### GIS and trajectory analysis

- GeoPandas
- Shapely
- PyProj
- QGIS
- MovingPandas
- OpenDrift
- OpenOil
- xarray
- NetCDF4
- SciPy

### Backend and database

- FastAPI
- PostgreSQL
- PostGIS
- SQLAlchemy
- GeoAlchemy2
- Redis, optional
- Celery or APScheduler for background jobs

### Frontend and deployment

- Streamlit for prototype dashboard
- React + TypeScript for final dashboard
- Leaflet or MapLibre GL for interactive maps
- Plotly or Chart.js for analytics charts
- Docker
- Docker Compose
- GitHub Actions, optional
- Linux/cloud VM, optional

---

## Database Design

```text
PostgreSQL + PostGIS
│
├── satellite_scenes
├── sar_processing_jobs
├── spill_detections
├── spill_polygons
├── environmental_grids
├── drift_runs
├── drift_particles
├── ais_messages
├── vessels
├── vessel_tracks
├── vessel_anomalies
├── attribution_scenarios
├── suspect_scores
└── investigation_reports
```

### Important tables

#### `spill_detections`

```text
id
scene_id
satellite_time_utc
detection_confidence
lookalike_risk
area_km2
perimeter_km
centroid
orientation_degrees
geometry
created_at
```

#### `ais_messages`

```text
id
mmsi
imo_number
vessel_name
vessel_type
timestamp_utc
latitude
longitude
speed_over_ground
course_over_ground
heading
navigational_status
geometry
```

#### `drift_runs`

```text
id
spill_id
simulation_type              # backward_hindcast / forward_forecast / vessel_scenario
start_time_utc
end_time_utc
wind_dataset
current_dataset
particle_count
time_step_minutes
uncertainty_configuration
output_geometry
```

#### `attribution_scenarios`

```text
id
spill_id
mmsi
release_time_utc
release_position
simulated_slick_geometry
overlap_iou
centroid_error_km
orientation_difference
drift_match_score
```

#### `suspect_scores`

```text
id
spill_id
mmsi
proximity_score
time_match_score
route_alignment_score
behaviour_anomaly_score
ais_gap_score
drift_match_score
final_score
rank
explanation
```

---

## API Design

### Spill detection APIs

```text
POST /api/v1/spills/detect
GET  /api/v1/spills
GET  /api/v1/spills/{spill_id}
GET  /api/v1/spills/{spill_id}/geometry
```

### Drift simulation APIs

```text
POST /api/v1/spills/{spill_id}/hindcast
POST /api/v1/spills/{spill_id}/forecast
GET  /api/v1/spills/{spill_id}/drift-runs
```

### AIS and vessel APIs

```text
POST /api/v1/ais/import
GET  /api/v1/spills/{spill_id}/vessels
GET  /api/v1/vessels/{mmsi}
GET  /api/v1/vessels/{mmsi}/track
GET  /api/v1/vessels/{mmsi}/anomalies
```

### Scenario and attribution APIs

```text
POST /api/v1/spills/{spill_id}/scenarios/run
GET  /api/v1/spills/{spill_id}/candidates
GET  /api/v1/spills/{spill_id}/candidates/{mmsi}
GET  /api/v1/spills/{spill_id}/ranking
```

### Dashboard APIs

```text
GET /api/v1/spills/{spill_id}/map-layers
GET /api/v1/spills/{spill_id}/report
GET /api/v1/spills/{spill_id}/export/geojson
GET /api/v1/spills/{spill_id}/export/csv
```

---

## Suggested Project Structure

```text
marine-oil-spill-attribution/
│
├── README.md
├── requirements.txt
├── docker-compose.yml
├── .env.example
│
├── data/
│   ├── raw/
│   │   ├── sentinel1/
│   │   ├── ais/
│   │   ├── wind/
│   │   └── currents/
│   ├── processed/
│   └── sample/
│
├── notebooks/
│   ├── 01_sar_preprocessing.ipynb
│   ├── 02_oil_slick_detection.ipynb
│   ├── 03_ais_analysis.ipynb
│   ├── 04_drift_hindcast.ipynb
│   └── 05_scenario_attribution.ipynb
│
├── models/
│   ├── segmentation/
│   ├── checkpoints/
│   └── configs/
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   ├── core/
│   │   ├── database/
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── services/
│   │   │   ├── sar_processor.py
│   │   │   ├── oil_detector.py
│   │   │   ├── spill_characterizer.py
│   │   │   ├── drift_engine.py
│   │   │   ├── ais_processor.py
│   │   │   ├── scenario_engine.py
│   │   │   └── ranking_engine.py
│   │   └── workers/
│   └── Dockerfile
│
├── frontend/
│   ├── streamlit_app.py
│   └── react-dashboard/
│
├── scripts/
│   ├── download_satellite_data.py
│   ├── import_ais_data.py
│   ├── run_detection.py
│   ├── run_hindcast.py
│   └── run_vessel_scenarios.py
│
├── tests/
│   ├── test_detection.py
│   ├── test_drift.py
│   ├── test_ais.py
│   └── test_ranking.py
│
└── docs/
    ├── architecture.md
    ├── api.md
    ├── data_dictionary.md
    └── methodology.md
```

---

## Installation

### Prerequisites

```text
Python 3.10+
PostgreSQL 14+
PostGIS 3+
Docker and Docker Compose, recommended
Git
ESA SNAP, optional but recommended for SAR preprocessing
```

### Clone repository

```bash
git clone [https://github.com/your-username/marine-oil-spill-attribution.git](https://github.com/your-username/marine-oil-spill-attribution.git)
cd marine-oil-spill-attribution
```

### Create virtual environment

```bash
python -m venv venv
```

#### Windows

```bash
venv\Scripts\activate
```

#### Linux/macOS

```bash
source venv/bin/activate
```

### Install Python dependencies

```bash
pip install -r requirements.txt
```

### Configure environment variables

```bash
cp .env.example .env
```

Example `.env`:

```text
DATABASE_URL=postgresql://postgres:password@localhost:5432/oilspill_db
REDIS_URL=redis://localhost:6379/0
MODEL_PATH=models/checkpoints/oil_slick_unet.pt
DATA_DIR=data/
ENVIRONMENT=development
```

### Start database and services

```bash
docker compose up -d
```

### Start backend

```bash
uvicorn backend.app.main:app --reload
```

### Start prototype dashboard

```bash
streamlit run frontend/streamlit_app.py
```

---

## Example End-to-End Run

```bash
# 1. Import Sentinel-1 scene metadata and imagery
python scripts/download_satellite_data.py --scene-id <SCENE_ID>

# 2. Preprocess SAR image
python scripts/preprocess_sar.py --scene-id <SCENE_ID>

# 3. Detect possible oil slick
python scripts/run_detection.py --scene-id <SCENE_ID>

# 4. Import AIS records for relevant period
python scripts/import_ais_data.py --start <UTC_TIME> --end <UTC_TIME>

# 5. Run backward hindcast from detected slick
python scripts/run_hindcast.py --spill-id <SPILL_ID> --hours 24

# 6. Run vessel-specific forward spill scenarios
python scripts/run_vessel_scenarios.py --spill-id <SPILL_ID>

# 7. Generate ranked candidate vessel list
python scripts/rank_vessels.py --spill-id <SPILL_ID>
```

---

## Evaluation Metrics

### Oil-slick segmentation

| Metric | Purpose |
|---|---|
| IoU / Jaccard score | Overlap between predicted and ground-truth oil mask |
| Dice score | Segmentation overlap quality |
| Precision | Reduces false oil-slick detections |
| Recall | Reduces missed spill detections |
| F1 score | Balances precision and recall |
| False-positive rate | Measures false alarms caused by look-alikes |

### Drift simulation

| Metric | Purpose |
|---|---|
| Centroid error | Distance between predicted and observed slick centroids |
| Polygon IoU | Overlap between simulated and observed slick polygons |
| Particle containment rate | Percentage of particles inside observed slick area |
| Area difference | Difference between predicted and observed slick area |
| Orientation difference | Difference in predicted and observed slick directions |

### Vessel attribution

| Metric | Purpose |
|---|---|
| Top-1 accuracy | Correct source appears at rank 1 in validated cases |
| Top-3 recall | Correct source appears in top three candidates |
| Mean Reciprocal Rank | Measures ranking quality |
| Drift-match score | Measures scenario-to-observation compatibility |
| Explanation completeness | Ensures every ranking score has visible evidence |

---

## Dashboard Features

The dashboard provides the following layers and panels:

```text
Map Layers
├── Sentinel-1 SAR image
├── Detected oil-slick polygon
├── Spill confidence and characteristics
├── Backward probable-origin region
├── Forward drift forecast
├── Historic AIS vessel tracks
├── Candidate vessel positions
├── Simulated spill polygons for candidate vessels
├── Coastline and marine-protected-area layers
└── Ports and shipping-lane layers
```

```text
Panels
├── Incident summary
├── Spill geometry and confidence
├── Drift time slider
├── Candidate vessel ranking table
├── Vessel-specific AIS timeline
├── Speed and course anomaly chart
├── Scenario simulation comparison
├── Evidence explanation panel
└── Incident report export
```

---

## Limitations

This project is an investigation-support system and has important limitations:

- A dark region in SAR imagery is not always oil.
- SAR-only detection cannot reliably determine oil thickness or total spill volume.
- AIS data may contain gaps, delayed messages, inaccurate information, or disabled transmissions.
- Ocean currents and wind forecasts contain uncertainty.
- Satellite revisit intervals may delay observation of a spill.
- Optical imagery may be unavailable because of cloud cover or nighttime conditions.
- A high vessel score is not proof of legal responsibility.
- Final attribution requires expert review, additional evidence, field verification, and legal investigation procedures.

---

## Future Enhancements

- Multi-temporal tracking across several SAR scenes.
- Integration of Sentinel-2, thermal imagery, and hyperspectral data.
- SAR ship detection to identify non-AIS vessels.
- Vessel-type and cargo-risk modelling.
- Deep-learning model for oil vs look-alike classification.
- Ensemble drift simulation using multiple ocean-current datasets.
- Automatic uncertainty quantification.
- Alerting system for coastal authorities.
- Mobile-responsive dashboard.
- PDF investigation-report generation.
- Model monitoring and active-learning feedback loop.
- Satellite tasking and real-time ingestion pipeline.
- Integration with port, offshore-platform, and shipping-lane databases.

---

## Ethical and Operational Use

This software should be used to prioritize investigation and response actions.

It must not be used as the only basis for:

- Legal prosecution
- Financial penalties
- Public accusation
- Vessel detention
- Publication of unverified allegations

All vessel rankings must be reviewed by qualified maritime, environmental, and legal authorities before action is taken.

---

## Project Status

```text
Current Stage: Prototype / Research System

Implemented / Planned Modules:
[ ] Sentinel-1 SAR preprocessing
[ ] Oil-slick segmentation model
[ ] Spill polygon extraction
[ ] Backward drift hindcasting
[ ] Forward spill forecasting
[ ] Historic AIS ingestion
[ ] Vessel trajectory reconstruction
[ ] AIS anomaly detection
[ ] Scenario-based vessel simulation
[ ] Explainable vessel ranking
[ ] GIS dashboard
[ ] Incident report export
```

---

## Disclaimer

The project provides an AI-assisted, geospatial, and physics-informed analytical workflow for oil-spill investigation. It produces probable spill detections, drift paths, and ranked vessel candidates. Its results are advisory and must be verified by domain experts before operational, legal, or enforcement decisions are made.
README improvements
Before publishing, replace these placeholders:

your-username in the GitHub clone command.

<SCENE_ID>, <SPILL_ID>, and <UTC_TIME> with your real input examples.

Current Stage checkboxes according to your actual implementation.

Add screenshots of:

SAR image before/after processing

Detected oil-slick mask

Backward drift map

AIS vessel tracks

Scenario comparison: observed versus simulated slick

Final vessel-ranking dashboard

Most importantly, include one visual showing red = observed satellite slick and purple/blue = simulated slick from candidate vessel. That image will make your core innovation immediately understandable.****
