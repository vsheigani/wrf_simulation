# Mesoscale Atmospheric Simulation - WRF

A complete [WRF (Weather Research and Forecasting)](https://www.mmm.ucar.edu/models/wrf) project covering the full simulation pipeline: WPS domain configuration and preprocessing, WRF model setup and execution, and Python-based post-processing using [Salem](https://salem.readthedocs.io/) and Matplotlib. The companion document [`wrf_paper.pdf`](wrf_paper.pdf) contains all instructions necessary for setting up the simulation environment (compiling WRF and WPS, environment variables, dependencies), configuring the model (namelists, domain nesting, physics options), running the preprocessing steps (geogrid, ungrib, metgrid), executing the simulation, and reproducing the post-processing results presented here.

## Project Overview

This project simulates two days of atmospheric conditions over the **Avalon Peninsula / North Atlantic** region (September 4–6, 2019) using the ARW (Advanced Research WRF) solver. It reads model output from a nested inner domain (d02, 4 km resolution) and produces publication-quality maps of:

| Variable | Description |
|---|---|
| 2-m Temperature | Surface air temperature with wind vectors |
| Sea Surface Temperature (SST) | Ocean skin temperature with wind vectors |
| Surface Pressure | PSFC with isobar contours |
| Mid-tropospheric Pressure | Pressure at model level 15 (~500 hPa) |
| Water Vapor Mixing Ratio | QVAPOR at model level 10 |

Output graphs are saved as PDF files in `graphs/`.

## WRF Background

### What is WRF?

The **Weather Research and Forecasting (WRF)** model is a mesoscale numerical weather prediction (NWP) system developed by NCAR, NCEP, and collaborating institutions. It solves the fully compressible, non-hydrostatic Euler equations on a terrain-following sigma-pressure hybrid vertical coordinate.

### Key Concepts

**Governing equations.** WRF solves the flux-form Euler equations in a mass coordinate (η). The prognostic variables include:

- Perturbation dry-air mass per unit area (μ')
- Horizontal momentum (U = μu, V = μv)
- Vertical momentum (W = μw)
- Potential temperature (Θ = μθ)
- Geopotential (φ)
- Mixing ratios for moisture species (Qm = μqm)

**Nested domains.** WRF supports one-way and two-way nesting. This simulation uses two domains:
- **d01**: parent domain, 12 km horizontal resolution, 57 × 52 grid points
- **d02**: inner nest, 4 km horizontal resolution (ratio 1:3), 139 × 124 grid points

Both domains use 45 vertical levels up to 5 hPa.

**Physics suite.** This simulation uses the `CONUS` physics suite, which bundles:
- Thompson microphysics
- Kain-Fritsch cumulus (outer domain) / none (inner domain)
- RRTMG longwave & shortwave radiation
- MYJ planetary boundary layer
- Noah land-surface model

**Map projection.** The domains are configured on a **polar stereographic** projection centred at 47.572°N, 52.734°W (offshore Newfoundland), which preserves shapes at high latitudes and is standard for mid-latitude NWP.

**Dynamics options used.**
- Non-hydrostatic mode (`non_hydrostatic = .true.`)
- Hybrid vertical coordinate (`hybrid_opt = 2`)
- Rayleigh damping in the upper atmosphere (`damp_opt = 3`)
- 3D Smagorinsky diffusion (`diff_opt = 2`, `km_opt = 4`)

### WRF Workflow

```
Real-world data (GFS analysis)
         │
   WPS (geogrid → ungrib → metgrid)
         │  creates met_em files
   WRF (real.exe → wrf.exe)
         │  produces wrfout_d0* NetCDF files
   This repo  ←─────────────────────────
   (Salem + Matplotlib visualisation)
```

## Setup

### Prerequisites

- Python ≥ 3.12
- [uv](https://docs.astral.sh/uv/) package manager

### Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Install Git LFS

The WRF output file (`results/wrfout_d02_selcomb022.nc`) is large and stored in this repository using [Git Large File Storage (LFS)](https://git-lfs.com/). You must install Git LFS before cloning so the actual file is downloaded instead of a pointer.

**macOS (Homebrew):**
```bash
brew install git-lfs
```

**Ubuntu/Debian:**
```bash
sudo apt install git-lfs
```

**Windows:** Download the installer from [git-lfs.com](https://git-lfs.com/).

After installing, activate LFS for your user account (only needed once per machine):

```bash
git lfs install
```

### Clone and install

```bash
git clone https://github.com/your-username/wrf.git
cd wrf

# Create virtual environment and install all dependencies
uv sync
```

`uv sync` reads `pyproject.toml` and `uv.lock`, creates a `.venv/`, and installs exact pinned versions of all dependencies — no manual `pip install` needed.

### Dependencies

| Package | Purpose |
|---|---|
| `salem` | Open WRF NetCDF files and handle their curvilinear grid / projection |
| `numpy` | Array operations on model fields |
| `matplotlib` | Plotting and PDF export |
| `geopandas` | Country/coastline boundaries for map backgrounds |
| `shapely` | Geometry operations (used by geopandas/salem) |
| `jupyter` / `notebook` | Interactive notebook environment |

## Usage

### Run the Jupyter notebook

```bash
uv run jupyter notebook wrf_end.ipynb
```

The notebook expects the WRF output file at:

```
results/wrfout_d02_selcomb022.nc
```

Graphs are written to `graphs/` as PDF files.

### WRF input files

The `wrf_input_files/` directory contains the namelist configuration used to run the WRF simulation:

- `namelist.wps` — WPS preprocessing configuration (domain, projection, input data)
- `namelist.input` — WRF run configuration (physics, dynamics, time control)

## Repository Structure

```
wrf/
├── wrf_end.ipynb          # Post-processing notebook
├── wrf_input_files/
│   ├── namelist.wps       # WPS namelist
│   └── namelist.input     # WRF namelist
├── results/
│   └── wrfout_d02_*.nc    # WRF output (tracked via Git LFS)
├── graphs/                # Generated figures (PDF)
├── pyproject.toml
└── uv.lock
```

## References

- Skamarock, W. C., et al. (2019). *A Description of the Advanced Research WRF Model Version 4*. NCAR Tech. Note NCAR/TN-556+STR. [doi:10.5065/1dfh-6p97](https://doi.org/10.5065/1dfh-6p97)
- Maussion, F., et al. (2017). *Salem: a Python library for geoscientific data processing*. [salem.readthedocs.io](https://salem.readthedocs.io/)
- WRF Users' Guide: [www2.mmm.ucar.edu/wrf/users/docs](https://www2.mmm.ucar.edu/wrf/users/docs/user_guide_v4/contents.html)
