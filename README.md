# RadCLss: Extracted Radar Columns and In-Situ Sensors

[![PyPI version](https://badge.fury.io/py/radclss.svg)](https://badge.fury.io/py/radclss)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)

RadCLss is a Python package for extracting vertical radar columns above surface instrumentation sites and collocating them with in-situ ground-based sensors. It is developed and maintained by the [Atmospheric Radiation Measurement (ARM) User Facility](https://www.arm.gov) to support precipitation process research.

## Overview

RadCLss combines scanning radar data (processed via [Py-ART](https://arm-doe.github.io/pyart/) CMAC) with collocated surface instrumentation to produce a unified time-height dataset. This enables direct comparison of radar-retrieved quantities (e.g., reflectivity, rain rate) against surface measurements of precipitation, particle size distributions, and meteorological state.

### Supported Instruments

| Key Prefix | Instrument | Description |
|---|---|---|
| `radar` | CMAC-processed radar | Scanning ARM precipitation radar files |
| `met` | Surface Meteorology | Temperature, pressure, humidity, wind |
| `sonde` | Radiosonde | Upper-air thermodynamic profiles |
| `pluvio` | Pluvio Weighing Gauge | Precipitation accumulation and rate |
| `ld` | Laser Disdrometer (LDQUANTS) | Particle size distribution |
| `vd` | Laser Disdrometer (VDISQUANTS) | Particle size distribution (supplemental) |
| `wxt` | WXT Weather Station | Compact meteorological sensor |

## Installation

### From PyPI

```bash
pip install radclss
```

### From Source

```bash
git clone https://github.com/ARM-Development/radclss.git
cd radclss
pip install -e .
```

### Conda Environment (Recommended)

```bash
conda env create -f continuous-integration/environment-actions.yml
conda activate radclss_env
pip install -e .
```

## Quick Start

```python
import radclss
import glob
from dask.distributed import Client, LocalCluster

# Build the volumes dictionary with paths to each instrument's files
volumes = {
    "date": "20250619",
    "radar": glob.glob("/data/bnf/bnfcsapr2cmacS3.c1/*20250619*.nc"),
    "sonde": glob.glob("/data/bnf/bnfsondewnpnM1.b1/*20250619*.cdf"),
    "met_M1": glob.glob("/data/bnf/bnfmetM1.b1/*20250619*"),
    "pluvio_M1": glob.glob("/data/bnf/bnfwbpluvio2M1.a1/*20250619*.nc"),
    "ld_M1": glob.glob("/data/bnf/bnfldquantsM1.c1/*20250619*.nc"),
}

# Define site locations: {site_name: (lat, lon, alt_m)}
input_site_dict = {
    "M1": (34.34525, -87.33842, 293),
    "S20": (34.65401, -87.29264, 178),
    "S30": (34.38501, -86.92757, 183),
}

# Run RadCLss with parallel processing
with Client(LocalCluster(n_workers=4, threads_per_worker=1)):
    ds = radclss.core.radclss(volumes, input_site_dict, serial=False, verbose=True)

# Save the output to ARM-formatted NetCDF
radclss.io.write_radclss_output(ds, "radclss_output.nc", "csapr2radclss.c2")

# Visualize extracted columns
fig, axarr = radclss.vis.create_radclss_columns("radclss_output.nc")
fig.savefig("radclss_columns.png", dpi=150, bbox_inches="tight")
```

## API Reference

### `radclss.core.radclss`

The primary processing function. Extracts radar columns above each site and matches in-situ sensor data.

```python
ds = radclss.core.radclss(
    volumes,           # dict: instrument file lists keyed as "<instrument>_<site>"
    input_site_dict,   # dict: site locations as {name: (lat, lon, alt_m)}
    serial=True,       # bool: use serial (True) or parallel Dask (False) processing
    dod_version="",    # str: ARM Data Object Description version (empty = latest)
    discard_var={},    # dict: variables to drop from each datastream
    verbose=False,     # bool: print progress information
    base_station="M1", # str: site used for the output time dimension
    current_client=None,  # dask.distributed.Client: existing Dask client
)
```

Returns an `xarray.Dataset` with dimensions `(time, height, station)` containing both radar-retrieved and in-situ variables.

### `radclss.io.write_radclss_output`

Writes the RadCLss dataset to an ARM-compliant NetCDF file, fetching encoding information from the ARM Data Object Description (DOD) API.

```python
radclss.io.write_radclss_output(
    ds,               # xarray.Dataset: RadCLss output dataset
    output_filename,  # str: output file path
    process,          # str: ARM datastream name (e.g., "csapr2radclss.c2")
    version=None,     # str: DOD version (None = latest)
)
```

### `radclss.vis.create_radclss_columns`

Generates a multi-panel figure showing the extracted radar column for each site.

```python
fig, axarr = radclss.vis.create_radclss_columns(
    radclss,           # str or xr.Dataset: path to RadCLss file or dataset
    field="corrected_reflectivity",  # str: radar field to display
    vmin=-5,           # float: colorbar minimum
    vmax=65,           # float: colorbar maximum
    stations=None,     # list[str]: subset of stations to plot (None = all)
)
```

### `radclss.vis.create_radclss_rainfall_timeseries`

Generates a three-panel timeseries figure comparing radar reflectivity, precipitation rate, and accumulated precipitation from radar and surface sensors.

```python
fig, axarr = radclss.vis.create_radclss_rainfall_timeseries(
    radclss,           # str or xr.Dataset: path to RadCLss file or dataset
    field="corrected_reflectivity",
    dis_site="M1",     # str: station to display
    rheight=750,       # int: height (m) for rain rate comparison
)
```

## Dependencies

- [Py-ART](https://arm-doe.github.io/pyart/) (`arm_pyart`) — Radar data processing
- [ACT](https://arm-doe.github.io/ACT/) (`act-atmos`) — ARM data I/O and utilities
- [xarray](https://xarray.dev/) — Labeled multi-dimensional arrays
- [xradar](https://docs.openradarscience.org/projects/xradar/) — Radar data in xarray format
- [Dask](https://www.dask.org/) — Parallel and out-of-core computing
- [NumPy](https://numpy.org/), [pandas](https://pandas.pydata.org/), [Matplotlib](https://matplotlib.org/)

## Contributing

Contributions are welcome. Please open an issue or pull request on the [GitHub repository](https://github.com/ARM-Development/radclss).

## License

RadCLss is released under the [MIT License](LICENSE).

## Acknowledgements

RadCLss is developed by the Atmospheric Radiation Measurement (ARM) User Facility, a U.S. Department of Energy Office of Science user facility managed by the Office of Biological and Environmental Research.
