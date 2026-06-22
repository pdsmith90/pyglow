# pyglow + IRI-2020 Integration Guide

This document describes how pyglow was configured to use the locally installed IRI-2020 model.

## Overview

pyglow natively supports IRI-2012 and IRI-2016 via compiled FORTAN extensions (`iri12py` and `iri16py`).
This integration adds IRI-2020 as `iri2020py`, following the same pattern.

## Steps Taken

### 1. Downloaded IRI-2020 to data/IRI/

The IRI-2020 model was downloaded from `https://irimodel.org/IRI-2020/` to `data/IRI/`:

- **FORTRAN source files:** `irisub.for`, `irifun.for`, `iriflip.for`, `iridreg.for`, `iritec.for`, `rocdrift.for`, `cira.for`, `igrf.for`, `iritest.for`, `irirtam.for`
- **Coefficient files:** All `dgrf*.dat` (1945–2020), `igrf202*.dat`, `MCSAT*.dat`, `CCIR*.dat`, `URSI*.dat`
- **Index files:** `ig_rz.dat` and `apf107.dat` (downloaded from ECHAIM: `https://chain-new.chain-project.net/echaim_downloads/`)

The `00_iri.zip` was extracted for complete CCIR/URSI/MCSAT data files.

### 2. Compiled the IRI-2020 Model

Compiled with gfortran using these flags: `-std=legacy -w -O2 -fbacktrace -fno-automatic -fPIC`

**Fixes applied:**
- `irifun.for`: Changed `SPHARM_IK` subroutine's `C(1000)` to `C(*)` (assumed-size) so callers can pass arrays of any size. Updated all 4 call sites to pass the correct size parameter.
- `igrf.for`: Enlarged `P(8,100)` to `P(8,3500)` to fix array bounds overflow in the field line tracing loop.

### 3. Configured pyglow

Created the following files/directories under `pyglow/src/pyglow/`:

#### a. `models/dl_models/iri2020/` — Data directory

Contains all IRI-2020 source files and data files. All files were copied from `data/IRI/` and `data/IRI/IRI-zip/`.

#### b. `models/f2py/iri2020/` — Build directory

Contains:
- `Makefile` — Build automation
- `sig.patch` — Patches the f2py-generated signature file with `intent(out)` and `intent(inplace)` annotations
- `iridreg.patch` — Fixes encoding issues in `iridreg.for`
- `delete_iriflip_comments.py` — Strips problematic Fortran comments from `iriflip.for`

Build process:
1. `python3 delete_iriflip_comments.py` → creates `iriflip_modified.for`
2. Copy `iridreg.for` → `iridreg_modified.for`
3. `f2py -m iri2020py -h sig_file.pyf *.for only: iri_sub read_ig_rz readapf107 :` → generates signature
4. Apply `sig.patch` → `sig_file_patched.pyf`
5. `f2py -c sig_file_patched.pyf *.for` → compiles `iri2020py.cpython-313-x86_64-linux-gnu.so`
6. Move `.so` to `pyglow/src/pyglow/`

#### c. `setup.py` — Updated

- Added `iri2020` Extension definition with all source files (no `cosd_sind.for` needed for IRI-2020)
- Added all data files to `data_files` under `'pyglow/iri2020_data/'`
- Added `'pyglow_trash', ['src/pyglow/models/dl_models/iri2020/dummy.txt']` for completeness

#### d. `src/pyglow/iri.py` — Updated

- Added imports for `iri2020py` (with renamed functions to avoid naming conflicts)
- Added `__INIT_IRI2020 = False` global flag
- Added `IRI.init_iri2020()` static method
- Added `version == 2020` case in `IRI.run()` that maps to `iri2020_data/` and `iri2020`

**Important:** pyglow's `IRI.run()` uses `os.chdir()` to the IRI data directory before calling the model. The code checks `'src'` in the path to override to `dl_models/iri2020` (development mode).

#### e. `src/pyglow/models/get_models.py` — Updated

- Added `model_urls_2020` for downloading `00_iri.zip` from `https://irimodel.org/IRI-2020/`
- Added download/unzip logic for IRI-2020

### 4. Fixed Data Files

Several `.dat`/`.asc` files returned 406/404 errors when downloaded without a User-Agent header.
The `00_iri.zip` from `https://irimodel.org/IRI-2020/` contains all the correct data files.
Extract and copy to `models/dl_models/iri2020/`.

## Usage

```python
import sys
sys.path.insert(0, 'pyglow/src')

from iri2020py import iri_sub, read_ig_rz, readapf107
import numpy as np
import os

# Must change to the IRI2020 data directory before calling
os.chdir('pyglow/src/pyglow/models/dl_models/iri2020')

# Initialize once per session
read_ig_rz()
readapf107()

# Configure switches (same as IRI-2016/2020 recommended defaults)
jf = np.ones((50,), dtype=int)
jf[3] = 0   # B0,B1 model
jf[4] = 0   # foF2 model
jf[5] = 0   # Ni model
jf[20] = 0  # ion drift
jf[22] = 0  # Te_topside
jf[27] = 0  # spreadF
jf[28] = 0  # NeQuick off
jf[29] = 0
jf[32] = 0  # Auroral boundary
jf[34] = 0  # foE storm
jf[21] = 0  # ion densities in m^-3
jf[33] = 0  # turn messages off

# Output array (100 elements)
oarr = np.zeros((100,))

# Call IRI-2020
# Parameters: jf, jmag, lat, lon, year, -doy, hour, alt_start, alt_end, step, oarr
outf = iri_sub(jf, 0, 20.0, 100.0, 2024, -115, 12.0, 60.0, 300.0, 10.0, oarr)

# Extract results
ne = outf[0, 0] / 1e6        # electron density in 1/cm^3
tn = outf[1, 0]              # neutral temperature in K
ti = outf[2, 0]              # ion temperature in K
te = outf[3, 0]              # electron temperature in K
nmf2 = oarr[0] / 1e6         # NmF2 in 1/cm^3
hmf2 = oarr[1]               # hmF2 in km
```

## Using via pyglow IRI class

```python
import sys
sys.path.insert(0, 'pyglow/src')

from pyglow import IRI, LocationTime

# Initialize IRI2020
IRI.init_iri2020()

# Run with version=2020
iri = IRI()
loc = LocationTime(2024, 1, 15, 12.0, 20.0, 100.0, 100.0)  # year, month, day, hour, lat, lon, alt
iri.run(loc, version=2020)

print(f"Ne: {iri.ne:.2e} 1/cm^3")
print(f"Te: {iri.Te:.1f} K")
print(f"Ti: {iri.Ti:.1f} K")
print(f"NmF2: {iri.NmF2:.2e} 1/cm^3")
print(f"hmF2: {iri.hmF2:.1f} km")
```

## File Layout

```
pyglow/
├── src/
│   ├── pyglow/
│   │   ├── __init__.py
│   │   ├── iri.py              (updated: added version=2020 support)
│   │   ├── iri2020py*.so       (compiled extension)
│   │   └── models/
│   │       ├── dl_models/
│   │       │   └── iri2020/    (all IRI-2020 data files)
│   │       └── f2py/
│   │           └── iri2020/    (build artifacts: Makefile, sig.patch, etc.)
│   └── pyglow/models/
│       └── get_models.py       (updated: added IRI-2020 download URLs)
├── setup.py                    (updated: added iri2020 Extension + data_files)
└── IRI2020_INTEGRATION.md      (this file)
```

## Notes

- **Python version:** Requires numpy with distutils support (numpy<2.1 or install `numpy.distutils` backport)
- **Fortran compiler:** gfortran with `-std=legacy` flag required for F77 compatibility
- **f2py backend:** Python 3.13+ uses meson backend; for older Python, the legacy distutils backend is used
- **Working directory:** IRI2020 must be called from its data directory (contains `ig_rz.dat`, `apf107.dat`, etc.)
- **Version 2020 support:** Added alongside existing 2012/2016 — no breaking changes

## IRI-2020 Model Components

The IRI-2020 uses these sub-models (per the output):
- **Ne (plasmasphere):** OTSR2012
- **Ne (topside):** IRI-2001
- **FoF2:** CCIR-1967 / URSI models
- **hmF2:** Shubin-2015 / AMBT-2012
- **B0,B1:** Bil-2000
- **D region:** IRI1990 (IRIDREG)
- **Ion composition:** RBV-2010 / TBT-2015 (FLIP model)
- **Te:** TBT-2012 with PF10.7 dependency
- **Ti:** TBKS2021
- **Neutral atmosphere:** NRLMSIS-00 (CIRA)
- **Magnetic field:** IGRF-2020/2021/2025/2026/2031/2036/2041
- **Equatorial drift:** Fejer et al., 2008 (ROC)
