# Palm Oil EUDR Traceability & Deforestation-Risk Screening

> An end-to-end GIS workflow for EU Deforestation Regulation (EUDR) due diligence in a Nigerian palm-oil supply chain: field collection, supplier–plantation linkage, geometry checks, land-use/land-cover (LULC) vegetation-loss screening against the 31 December 2020 cut-off, and delivery as an ArcGIS Dashboard, a Leaflet web map and a GeoJSON export.

- **Live Demo (web map):** https://progress-akpan-portfolio.vercel.app/projects/eudr-palm-oil-traceability/map/ (mirror on [GitHub Pages](https://progress-peter.github.io/eudr-palm-oil-traceability-dashboard/))
- **Case study:** https://progress-akpan-portfolio.vercel.app/projects/eudr-palm-oil-traceability/
- **ArcGIS Dashboard:** https://sambusgeospatial.maps.arcgis.com/apps/dashboards/9c551a1a286e440797ef608c25b2c594

**Status:** Methodology demonstrator built at Sambus Geospatial Nigeria Limited. All supplier and plantation records are **synthetic**, created for the demo. The loss figures are outputs of the LULC change analysis run over those synthetic boundaries.

---

## Overview

Palm-oil buyers exporting to the EU must prove, plot by plot, that what they buy was not grown on land deforested after 31 December 2020. In Nigeria that is hard. Supply is dominated by smallholders, fruit and oil move through aggregators and millers, and plantation boundaries are rarely recorded.

This project shows how a standard Esri stack can close that gap:

1. Suppliers and plantations are captured in **ArcGIS Field Maps**.
2. They are linked in **ArcGIS Online** and checked in **ArcGIS Pro**.
3. Each plantation and its 10 km surroundings are screened for vegetation loss by comparing **2020 and 2025 LULC classifications**.
4. The results are published as an **ArcGIS Dashboard** and a **Leaflet.js web map**. The map reads the same public feature service and exports plantation polygons as GeoJSON.

## Problem statement

Regulation (EU) 2023/1115 requires palm oil placed on the EU market to be:

- **deforestation-free**, meaning it was not produced on land converted from forest after 31 December 2020;
- **geolocated**, with polygons for plots larger than 4 ha.

Nigerian supply chains usually meet neither requirement: plantation boundaries are undocumented, a single aggregator or mill sources from many smallholdings, some plantations have no traceable buyer at all, and there is no systematic way to flag land-cover change since the cut-off.

## Objectives

1. Design a field-collectable data model linking suppliers (aggregators and millers) to plantation polygons, which also supports standalone plantations.
2. Screen each plantation and its 10 km buffer for vegetation loss between 2020 and 2025 using LULC change.
3. Summarise loss area and loss share per plantation in an ArcGIS Dashboard for due-diligence review.
4. Publish an open web map, and export plantation geometries and screening attributes as GeoJSON.

## Study area

Oil-palm-producing parts of five southern Nigerian states: **Akwa Ibom, Cross River, Ebonyi, Edo and Imo**.

| State | Plantations | Suppliers |
|---|---|---|
| Akwa Ibom | 2 | 1 aggregator, 1 miller |
| Cross River | 5 | 1 aggregator, 1 miller |
| Ebonyi | 5 | 1 miller (4 plantations unlinked) |
| Edo | 2 | 1 miller |
| Imo | 3 | 1 miller |

Screening runs at two scales: inside each plantation polygon, and inside a 10 km buffer around it.

## Data & sources

| Dataset | Source | Date | Resolution | Format |
|---|---|---|---|---|
| Suppliers (7 points) | **Synthetic**, created for this demo | 2026 | — | Hosted feature layer |
| Plantations (17 polygons) | **Synthetic**, Field Maps schema | Collected Jul–Aug 2026 | — | Hosted feature layer |
| LULC, baseline | Own LULC classification | 2020 | Not yet documented | Classified raster |
| LULC, current | Own LULC classification | 2025 | Not yet documented | Classified raster |
| 10 km plantation buffers | Derived (ArcGIS Pro *Buffer*) | — | — | Hosted feature layer |
| State boundaries | Nigeria administrative boundaries | — | — | Hosted feature layer |
| Basemaps | Esri Light Gray Canvas, World Imagery, World Topographic | Current | Varies | Tile services |

The public feature service `EUDR_Traceability_Demo` on ArcGIS Online backs both the dashboard and the web map. Its layers are 0 Selected States, 1 10km Palm Plantation Buffer, 2 Plantations and 3 Suppliers. Rasters are not published.

## Methodology

```
Field Maps → ArcGIS Online (link suppliers ↔ plantations) → Geometry QA (Calculate Geometry)
   → LULC 2020 + LULC 2025 → vegetation-loss raster → Tabulate Area (plantation + 10 km buffer)
   → join → ArcGIS Dashboards + Leaflet web map → GeoJSON export
```

1. **Data model.** Suppliers are points with ID, name, type (Aggregator or Miller), state and annual volume. Plantations are polygons with ID, farmer ID, state, certification, evidence, verification, collection date and estimated supply. They link through a nullable `Supplier_ID` with `Supplier_Link_Status` set to Linked or Unlinked.
2. **Geometry verification.** `Area_ha` was recalculated from the geometry in a projected CRS. PLT-005 carried a placeholder 35.63 ha but measures about 1,008 ha, roughly 28 times larger.
3. **LULC classification.** The study area was classified for 2020 (the cut-off year) and 2025 with the same class scheme. (Imagery source, class list and classifier settings are still to be documented here.)
4. **Vegetation loss.** Any pixel in a vegetation class in 2020 that is no longer in a vegetation class in 2025 is counted as loss. Pixels still in a vegetation class in 2025 count as retained vegetation.
5. **Zonal summary.** *Tabulate Area* runs in a projected CRS, once by plantation and once by 10 km buffer:
   `NonLoss_ha = VALUE_0 / 10,000`, `Loss_ha = VALUE_1 / 10,000`, `Loss_Percent = Loss_ha / (Loss_ha + NonLoss_ha) × 100`.
6. **Publishing.** The tables are joined to the layers, published to ArcGIS Online and used in the ArcGIS Dashboard. The Leaflet map queries the service as GeoJSON (`f=geojson`, `outSR=4326`) at runtime.
7. **Wording.** Flags read "potential risk: requires further review", never "deforested".

## Results

- **Overall loss inside plantations:** 236 ha of the 51,482 ha mapped (0.46%). The maximum for any one plantation is 2.19%.
- **One plantation dominates:** PLT-069, a standalone Ebonyi plantation, accounts for 166 ha, about 70% of all loss inside plantations.
- **Buffer loss is higher than plot loss for 15 of 17 plantations.** It peaks at 3.77% around PLT-046 (Imo), 3.66% around PLT-021 (Edo) and 3.23% around PLT-020 (Edo).
- **Screening status follows evidence status:** the 2 "Potential Risk" plantations (PLT-014, PLT-068) are the 2 with *Review Required* evidence.

| Plantation | State | Supplier | Area (ha) | Loss inside (ha) | Loss inside | Loss, 10 km buffer | Screening |
|---|---|---|---|---|---|---|---|
| PLT-069 | Ebonyi | Unlinked | 7,555 | 165.5 | 2.19% | 0.75% | Low |
| PLT-046 | Imo | MIL-009 | 2,197 | 31.1 | 1.42% | 3.77% | Low |
| PLT-014 | Cross River | AGG-004 | 4,822 | 25.1 | 0.52% | 0.39% | Potential Risk |
| PLT-063 | Ebonyi | MIL-012 | 1,680 | 2.4 | 0.14% | 0.17% | Low |
| PLT-015 | Cross River | MIL-003 | 5,114 | 6.6 | 0.13% | 1.24% | Low |
| PLT-068 | Ebonyi | Unlinked | 1,417 | 1.6 | 0.11% | 0.51% | Potential Risk |

No accuracy figures are reported, because the classification and loss map have not been validated against field or reference data.

## Limitations

- **Synthetic supply-chain records.** The workflow is the deliverable; the specific flags are not.
- **Vegetation loss ≠ EUDR deforestation.** EUDR uses FAO forest criteria and excludes agricultural plantations. The "any vegetation class lost" rule is broader, so it over-flags. That makes it a first-pass screen only.
- **Two-date comparison.** It misses clearing followed by regrowth, and errors in either classification propagate into the loss figures.
- **No accuracy assessment yet.** There are no omission or commission estimates.
- **Overlapping buffers.** Buffer losses cannot be summed across plantations.
- **Live data dependency.** The web map reads the ArcGIS Online service at runtime. If its sharing changes, the map shows an error banner.
- **Next step:** cross-check flags against JRC Global Forest Cover 2020 and Hansen/GFW tree-cover loss.

## Technologies

ArcGIS Pro (Spatial Analyst) · ArcGIS Online · ArcGIS Field Maps · ArcGIS Dashboards · Leaflet · JavaScript · ArcGIS REST API · Vercel · GitHub Pages

## Reproducibility

**Requirements:** ArcGIS Pro 3.x with Spatial Analyst, and an ArcGIS Online organisational account.

1. Load the plantation polygons and supplier points into a file geodatabase (feature classes, not shapefiles, to keep full field names). Create 10 km buffers, then recalculate `Area_ha` with *Calculate Geometry* in a projected CRS.
2. Produce LULC rasters for 2020 and 2025 with the same class scheme and grid.
3. In *Raster Calculator*, build a binary loss raster: 1 where the 2020 class is a vegetation class and the 2025 class is not, 0 where the pixel is vegetation in both years.
4. Run *Tabulate Area* (projected CRS) with the plantations as zones, then again with the 10 km buffers. Add `NonLoss_ha`, `Loss_ha` and `Loss_Percent` with the formulas above, and join both tables back to the layers.
5. Publish the joined feature classes to ArcGIS Online and build the dashboard.
6. Web map: `docs/index.html` is static. Point the `SVC` constant at your feature service and host the folder on any static host (GitHub Pages: Settings → Pages → `main`, `/docs`). Preview locally with `python -m http.server -d docs 8000`.

```
eudr-palm-oil-traceability-dashboard/
├── README.md
├── docs/index.html          # Leaflet web map (reads the public feature service, GeoJSON export)
└── docs/vendor/leaflet/     # Leaflet 1.9.4 (BSD-2-Clause)
```

## Author

**Progress Peter Akpan**, GIS & remote sensing analyst, Lagos, Nigeria · [Portfolio](https://progress-akpan-portfolio.vercel.app)

**CV summary:** Built an EUDR palm-oil traceability demo in ArcGIS Pro, Field Maps, ArcGIS Online, ArcGIS Dashboards and Leaflet that links 17 plantations to 7 suppliers and screens them for 2020–2025 vegetation loss with LULC change detection; the screen found 236 ha of loss inside plantations, concentrated in one unlinked plot, and buffer loss above plot loss for 15 of 17 plantations.
