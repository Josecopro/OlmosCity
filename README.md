# Landslide Risk Analysis — Medellín

This repository contains a Jupyter Notebook-based workflow to analyze and map landslide risk for the Medellín study area using population locations, historical landslide events, elevation (DEM), and precipitation data.

The main notebook is `OlmosCity_V1.ipynb` and implements data ingestion, preprocessing, terrain analysis, risk scoring and a small cellular automaton to propagate spatial risk between neighboring population points. The notebook exports several artifacts including a risk GeoTIFF, alert CSV, and visualizations.

---

## Important notes

- The dataset files are NOT included in this repository because they were too large to upload. You must download them from the shared Drive that was provided to the project and place them in the `data/` folder of this repository (or update the paths in the notebook to point to their locations).
- If you run this project locally, please update the data file paths in the notebook to point to your local files. Example paths in the notebook currently point to `/content/...` which is for cloud environments; change them to `data/ASTGTMV003_N06W076_dem.tif`, `data/col_pd_2020_1km_ASCII_XYZ.csv`, `data/Inventario_de_movimientos_en_masa_6829763638631547229.csv`, and `data/Colombia+Antioquia_Monthly.csv` or your chosen locations.
- Remove the cell that contains the inline package installation (the one with `!pip install -q rasterio`) if you are running the notebook locally. Instead, install dependencies from the provided `requirements.txt`.

---

## What this project does

1. Loads population points, historical landslide inventory, a DEM (Digital Elevation Model) and precipitation tables.
2. Reads the DEM and converts it into points, which are interpolated onto a regular grid to compute slope.
3. Interpolates slope values at each population point and computes statistics (mean, std).
4. Computes three main risk components per population point:
   - Slope-based risk (categorical mapping of slope in degrees to risk probability)
   - Historical landslide proximity risk (exponential decay by distance)
   - Precipitation-based risk (annual precipitation mapped to risk categories)
   - A normalized elevation term is optionally included
5. Combines these components using configurable weights to obtain a combined risk score (0-1).
6. Runs a cellular automaton (small spatial propagation) for a few iterations to spread risk locally between nearby population points.
7. Identifies high-risk points, generates alerts (`landslide_alerts.csv`) and exports a continuous risk GeoTIFF and visualizations.

---

## Cellular Automaton

At the core of the spatial propagation step is a simple cellular automaton (CA) applied to population points (treated as cells). The CA spreads risk locally so that high-risk points tend to increase the risk of their neighbors.

- Grid: The CA operates on the set of population points (each point is a cell). There is no regular grid; neighborhood is defined by distance in geographic coordinates.
- Neighborhood: For each point, neighbors are found within a fixed radius (the notebook uses ~0.0015 degrees ≈ 150 m). A KD-tree (`scipy.spatial.cKDTree`) is used for efficient neighbor queries.
- State: Each cell's state is its current risk score (a float between 0 and 1).
- Update rule: For each iteration, the cell's new risk becomes:

  new_risk = min(current_risk + 0.10 \* mean(neighbor_risks), 1.0)

  That is, the cell gains 10% of the neighborhood mean risk (the code uses mean \* 0.10). The risk is clipped to a maximum of 1.0.

- Iterations: The notebook runs the automaton for a small number of iterations (configurable, default = 3). This provides local smoothing/propagation of high-risk clusters while limiting runaway growth.

Why this helps: Landslides are spatially clustered — a high-risk site increases the chance of nearby locations being unstable. The CA represents a lightweight spatial diffusion of risk that preserves local patterns detected by the primary risk components.

---

## Running locally (Windows / Linux / macOS)

1. Create a Python environment (recommended):

   - Windows (PowerShell):

     ```powershell
     python -m venv .venv; .\.venv\Scripts\Activate.ps1
     ```

   - macOS / Linux:

     ```bash
     python3 -m venv .venv; source .venv/bin/activate
     ```

2. Install dependencies from `requirements.txt`:

   ```bash
   pip install -r requirements.txt
   ```

3. Place the required data files into the `data/` folder or update paths inside `OlmosCity_V1.ipynb` (search for PATH\_\* variables). Stop and edit the notebook to remove any in-notebook `!pip install` cells.

4. Start Jupyter Lab / Notebook and run the notebook cells in sequence.

   ```bash
   jupyter lab
   ```

---

## File outputs

- `medellin_risk.tif` — continuous risk raster in EPSG:32618
- `landslide_alerts.csv` — high-risk point alerts
- `risk_heatmap.png`, `risk_distribution.png`, `slope_map.png` — visualization images
- `population_heatmap.html` — interactive folium map

---

## Caveats and tips

- The notebook uses interpolation (cubic) and KD-tree nearest-neighbors. With large point counts this can be memory and CPU heavy. Consider using coarser resolution settings or subsampling if you run into performance issues.
- The DEM used in the notebook is a SRTM/ASTER tile. Check its CRS before reprojecting. The script assumes EPSG:4326 input for the DEM extraction step.
- If you need reproducible runs on different machines, pin package versions in `requirements.txt` (already provided).

---

## Drive link

The raw data files were shared separately via Google Drive. Please use the link provided in the project communications to download the following files and place them in `data/`:

- ASTGTMV003_N06W076_dem.tif
- col_pd_2020_1km_ASCII_XYZ.csv
- Inventario_de_movimientos_en_masa_6829763638631547229.csv
- Colombia+Antioquia_Monthly.csv

(If you need the Drive link added to this README, paste it here and I will update the file.)

Google Drive link:

https://drive.google.com/drive/folders/1BuFpElD_WimLSMITAxGwoxkzQtrzC4hx?usp=sharing

Please download the files from Google Drive and place them inside the `data/` folder of your local or Colab repository. Alternatively, update the file paths in your notebook to match your local directory structure.
