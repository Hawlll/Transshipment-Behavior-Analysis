# Analysis Decision Log

For each decision, append another block below using the same heading and four paragraphs in order.

---

## 2026-03-25

EDA showed three Global Fishing Watch tables (encounters, loitering, vessel registry). Encounters captured fewer at-sea meetings, while loitering had many more rows and similar regional hotspots, which matched the question about lingering behavior, speed, and duration near protected areas.

We anchored the modeling pipeline on **loitering events** as the row-level unit, while still using vessel metadata for maps and labels. Encounters remained useful context in EDA but were not the primary table for MPA proximity and clustering.

We considered treating encounters and loitering as a combined event stream, but that would mix incompatible semantics (pairwise meeting vs. solo loitering) and inflate preprocessing complexity without a clear gain for the MPA-distance question.

The downstream engineering and modeling notebooks assume one processed loitering table (`processed-loitering-events.csv`) with one event per row.

---

## 2026-04-01

We needed a consistent way to attach each loitering event to a named MPA and a scalar “closeness” feature for modeling and dashboards.

We loaded **WDPA / WD-OECM April 2026 public marine** geodata from the FileGDB, used the **point** feature class, exploded multipart geometries to coordinates, and for each event took the MPA point with **minimum Euclidean distance** in latitude/longitude space to the event’s starting coordinate. We then stored name, representative lat/long, and a degree-based separation (`nearest_mpa_*` columns) and wrote the enriched table to `data/processed/transshipment/processed-loitering-events.csv`.

We considered using polygon geometries for true boundary distance or a projected metric CRS, and faster spatial indices (e.g. `BallTree` on radians) instead of per-row `argmin` over all MPA points; those were deferred for implementation simplicity given the point layer already gave a usable first-order proximity signal.

Nearest-MPA features are approximate (chord vs. geodesic; degree-based “distance”), so interpretation stays qualitative—ranking closeness rather than claiming exact nautical miles to a reserve boundary.

---

## 2026-04-15

Raw timestamps and a naive UTC night flag would misclassify vessels far from the prime meridian, and total duration alone does not describe spatial drift during an event.

We derived **`local_hour`** as UTC hour plus longitude/15 (wrapped), defined **`is_night_corrected`** as 18:00–06:00 local solar time, and computed **`movement_mag`** as the Euclidean displacement between starting and ending lat/lon for each loitering window.

We considered using end timestamps for time-of-day, or tighter astronomical night definitions; we kept a simple solar-hour proxy aligned with the project scope and notebook readability.

These fields became core inputs to PCA alongside duration, median speed, and `nearest_mpa_dist`.

---

## 2026-04-20

The feature set is mixed units and scales, and we wanted interpretable groups of vessels for profiling rather than a single black-box classifier.

We **standardized** the six modeling features (`total_event_duration`, `median_speed_knots`, `nearest_mpa_dist`, `movement_mag`, `local_hour`, `is_night_corrected`), applied **PCA** with **five** components after inspecting cumulative explained variance, then ran **KMeans** with **K = 6**, chosen where **silhouette score** peaked and the inertia curve showed an elbow (`RANDOM_SEED = 21`, `n_init = 10`).

We considered other cluster counts, density-based methods (e.g. DBSCAN), or UMAP coordinates as clustering input; we stayed with PCA + KMeans for stability, speed, and linearly interpretable loadings, using **UMAP only on the PCA embedding** for two-dimensional visualization.

Cluster assignments and PCA coordinates feed the Dash dashboard and the cluster interpretation tables in `03_modeling.ipynb`.

---

## 2026-04-20

After identifying cluster 1 as the “Nighttime Stealth Loitering” / high-risk profile in the PCA signature table, we needed evidence that its proximity to MPAs was not a clustering artifact.

We ran **10,000 permutations** that fixed cluster 1’s size but reassigned distances at random from the pooled event distances, compared the observed mean `nearest_mpa_dist` for cluster 1 to that null distribution, and applied a **Bonferroni** correction with **α = 0.05 / 6** clusters.

We considered fewer permutations or uncorrected multiple testing; we kept 10k for a stable tail probability and Bonferroni to acknowledge testing six clusters.

The resulting **p-value** is reported in the notebook output and summarized in the README as strong evidence against the null for that cluster’s mean proximity.

---

## 2026-04-21

Stakeholders reading only the README might treat any “high risk” cluster as proof of illegal fishing, which the data and methods cannot support.

We expanded the README with **limitations** (2017 GFW vintage, linear PCA, AI-curated source positions, no legal claim) and a **“Big Idea”** section that frames outputs as **prioritization / suspicion**, not adjudication.

We considered leaving findings only inside notebooks; we rejected that because reproducibility and responsible framing matter for coursework and external readers.

The public narrative now matches the notebook’s careful language while still highlighting the nighttime / MPA-border cluster as the main behavioral signal.

---

## [Date in YYYY-MM-DD]

[What you found that prompted the decision]

[What you decided and why]

[What you considered and rejected, and why]

[Effect on the dataset or model]

---
