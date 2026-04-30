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
uv run jupyter notebook wrf_postprocessing.ipynb
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

## Running WPS and WRF on Compute Canada (Nibi Cluster)

At the time of writing, the available versions on the Nibi cluster are `wps/4.6.0` and `wrf/4.6.1`. Check for the latest available versions with:

```bash
module spider wrf
module spider wps
```

### Loading precompiled modules and copying working directories

Load the modules and copy the pre-built WRF and WPS directories into your working location:

```bash
module load wrf/4.6.1
module load wps/4.6.0

cp -r $EBROOTWRF .
cp -r $EBROOTWPS .
```

### Running WPS

Set up the WPS configuration (domain, input data, variable tables) as described in [`wrf_paper.pdf`](wrf_paper.pdf) and the [WPS documentation](https://github.com/wrf-model/WPS). Once configured, submit a job script that runs the WPS executables using `srun`:

```bash
srun ./geogrid.exe
srun ./ungrib.exe
srun ./metgrid.exe
```

### Running WRF

After setting up the WRF experiment directory as described in [`wrf_paper.pdf`](wrf_paper.pdf), run `real.exe` to generate the initial and boundary condition files, then launch the simulation:

```bash
srun ./real.exe
srun ./wrf.exe
```

Both steps should be submitted via SLURM job scripts using your allocation account.

## References

- National Centre for Atmospheric Research. *User's Guide for Advanced Research WRF (ARW) Modeling System Version 2.2*. June 2007. url: https://homepages.see.leeds.ac.uk/~lecrrb/wrf/aRWUsersGuide.pdf (visited on 04/16/2024).
- George D. Greenwade. "The Comprehensive Tex Archive Network (CTAN)". In: *TUGBoat* 14.3 (1993), pp. 342–351.
- W. C. Skamarock et al. "A Description of the Advanced Research WRF Version 3". In: (2008). doi: [10.5065/D68S4MVH](https://doi.org/10.5065/D68S4MVH).
- University of Waterloo. *WRF Tutorial*. June 27, 2019. url: https://wiki.math.uwaterloo.ca/fluidswiki/index.php?title=WRF_Tutorial (visited on 04/16/2024).
- Louis J. Wicker and William C. Skamarock. "Time-Splitting Methods for Elastic Models Using Forward Time Schemes". In: *Monthly Weather Review* 130.8 (2002), pp. 2088–2097. doi: [10.1175/1520-0493(2002)130\<2088:TSMFEM\>2.0.CO;2](https://journals.ametsoc.org/view/journals/mwre/130/8/1520-0493_2002_130_2088_tsmfem_2.0.co_2.xml).
