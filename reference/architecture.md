# E3SM Architecture

E3SM is not a single program. It is a coupled system of seven possible
component models (atmosphere, land, ocean, sea ice, river, land ice, wave)
plus one coupler driver, all built and orchestrated through CIME (the
Community Infrastructure for Modeling the Earth). This page is the map you
need before you edit anything or open a case.

---

## High-level layout

```
E3SM/
├── cime/                     ← submodule: CIME case-control system (shared with CESM)
├── cime_config/              ← E3SM-specific compsets, grids, machine configs, test suites
├── components/               ← one subdirectory per component model
│   ├── eam/                  ← E3SM Atmosphere Model (Fortran, EAMv3)
│   ├── eamxx/                ← Next-gen atmosphere (C++/Kokkos, SCREAM)
│   ├── elm/                  ← E3SM Land Model
│   ├── mosart/               ← River routing
│   ├── mpas-ocean/           ← Ocean (MPAS framework, Voronoi mesh)
│   ├── mpas-seaice/          ← Sea ice (MPAS framework + Icepack column physics)
│   ├── mpas-albany-landice/  ← MALI land ice
│   ├── mpas-framework/       ← Shared MPAS infrastructure (mesh, IO, registry)
│   ├── homme/                ← High-Order Method Modeling Environment (spectral element dycore)
│   ├── cice/                 ← Legacy CICE sea ice (rarely used in v2/v3)
│   ├── ww3/                  ← WaveWatch III (optional wave component)
│   ├── data_comps/           ← Data component models (DATM, DOCN, DICE, ...)
│   ├── stub_comps/           ← Stub components (SATM, SLND, SOCN, ...)
│   └── xcpl_comps/           ← Test-only exchange components
├── driver-mct/               ← MCT-based coupler driver (default in v2/v3)
├── driver-moab/              ← MOAB-based coupler driver (newer, for unstructured-to-unstructured remapping)
├── share/                    ← Shared utilities (timing, RNG, streams, esmf interface)
└── externals/                ← Submodules: scorpio, ekat, mam4xx, mct, YAKL, cub, ninja
```

Every coupled simulation has exactly one model active in each of seven
**slots**: ATM, LND, ICE, OCN, ROF, GLC, WAV. The compset name encodes
which model goes in each slot (see `running-cases.md`).

---

## The seven coupling slots

| Slot | Active models in E3SM | Stub | Data |
|------|------------------------|------|------|
| ATM (atmosphere) | EAM, EAMXX (SCREAM) | SATM | DATM |
| LND (land) | ELM | SLND | DLND (rare) |
| ICE (sea ice) | MPASSI (MPAS-Seaice), CICE (legacy) | SICE | DICE |
| OCN (ocean) | MPASO (MPAS-Ocean) | SOCN | DOCN |
| ROF (river) | MOSART | SROF | DROF (rare) |
| GLC (land ice) | MALI | SGLC | (none) |
| WAV (wave) | WW3 | SWAV | (none) |

Stub components do nothing; data components read prescribed forcing from
files (e.g., `DATM` reads observed atmospheric forcing for an ELM-only
"I" case).

---

## EAM: E3SM Atmosphere Model (Fortran, EAMv3)

`components/eam/`

The production atmosphere through E3SM v3. Written in Fortran. Uses the
HOMME spectral-element dynamical core on a cubed-sphere mesh (e.g., `ne30`)
with the physics grid (`pg2`) for tracer transport and parameterized
physics. Key modules:

| Scheme | Purpose | Source |
|--------|---------|--------|
| HOMME | Spectral-element dycore | `components/homme/` (built into EAM) |
| ZM (Zhang-McFarlane) | Deep convection with dCAPE-ULL trigger and convective microphysics | `components/eam/src/physics/cam/zm_*` |
| P3 | Stratiform cloud microphysics, two-moment with predicted rime mass and volume | `components/eam/src/physics/cam/micro_p3*` |
| CLUBB | Subgrid turbulence and shallow clouds | `components/eam/src/physics/clubb/` |
| RRTMG | Radiation (long+short wave) | `components/eam/src/physics/rrtmg/` |
| MAM4 | Modal aerosol model, 4 modes | `components/eam/src/chemistry/modal_aero/` |
| MAM5 | 5-mode variant with stratospheric sulfate | as above |
| chemUCI / Linoz v3 | Tropospheric and stratospheric chemistry | `components/eam/src/chemistry/{mozart,pp_*}/` |
| OROGRAVITY/oro_drag | Orographic gravity wave drag and turbulent mountain stress | `components/eam/src/physics/cam/gw_*`, `tms.F90` |
| COSP | Satellite simulator for output diagnostics | `components/eam/src/physics/cosp2/` |
| VBS | Volatility basis set for secondary organic aerosols | `components/eam/src/chemistry/aerosol/` |

EAM physics docs: `components/eam/docs/tech-guide/` (one file per scheme).

User-facing namelists go in `user_nl_eam`. Output is controlled with
`fincl1..fincl6`, `nhtfrq`, `mfilt`, `avgflag_pertape` (see
`output-and-postprocess.md`).

---

## EAMxx (SCREAM): Next-gen atmosphere (C++/Kokkos)

`components/eamxx/`

A clean-start atmosphere model written in C++ with Kokkos for performance
portability across CPU, NVIDIA GPU, AMD GPU, and Intel GPU. **Almost no
code is shared with EAM.** Currently shipped only in the km-scale
explicit-convection configuration called **SCREAM** (Simple Cloud-Resolving
E3SM Atmosphere Model).

Key facts:

- Compset suffix `-SCREAMv1` selects EAMxx (e.g., `F2010-SCREAMv1`).
- Standalone testing without CIME via
  `components/eamxx/scripts/test-all-eamxx -m <MACHINE>`.
- Uses RRTMGP (not RRTMG) for radiation, P3 (shared concept with EAM but
  separate C++ implementation), SHOC for turbulence, and HOMMEXX (a C++
  port of HOMME) for the dycore.
- Aerosols via MAM4xx (`externals/mam4xx`), the C++ port of MAM4.
- Build types: `sp` (single precision), `dbg` (debug), `fpe` (floating
  point exceptions on), `opt` (optimized).

EAMxx docs: `components/eamxx/docs/{user,developer,technical}/`.

For AI agents editing under `components/eamxx/**`, see
`components/eamxx/AGENTS.md` for standalone testing instructions.

---

## ELM: E3SM Land Model

`components/elm/`

Land-surface model. Originally forked from CLM4.5 (NCAR's Community Land
Model) and now diverged. ELM tracks the same prognostic state vector
philosophy (soil temperature and moisture profiles, snow layers, vegetation
PFTs, biogeochemistry), but the source tree, bug-fix history, and physics
options have evolved independently from CLM/CTSM.

Notable in-tree externals (under `components/elm/src/external_models/`):

| External | Purpose |
|----------|---------|
| `fates` | Functionally Assembled Terrestrial Ecosystem Simulator (vegetation demography) |
| `mpp` | Multi-Physics Problem solver (subsurface hydrology and thermodynamics) |
| `sbetr` | Subsurface BeTR biogeochemical transport |

Compsets:
- `I*` series: land-only (with `DATM` data atmosphere). Examples:
  `I1850CNPRDCTCBCTOP`, `I20TRCNPRDCTCBCTOP`, `IELMBC`.
- The land component is also active inside every coupled `WCYCL*` and `F*`
  compset.

ELM docs: `components/elm/docs/{user-guide,tech-guide,dev-guide}/`.

---

## ELM vs CTSM (CLM5)

A frequent point of confusion: the Community Terrestrial Systems Model
(CTSM, CLM5+) at NCAR and ELM at DOE share a common ancestor (CLM4.5,
~2014) but are **not** the same code. Bug fixes do not flow automatically
between them. New physics options published as "CLM5" or "CTSM5" need to
be ported deliberately to be available in ELM.

If you need a CLM/CTSM-specific feature in E3SM, check `components/elm/src/`
first; the option may have been ported under a different namelist switch.
If not, the porting effort is significant and should be coordinated through
the ELM team.

---

## MPAS-Ocean

`components/mpas-ocean/`

The ocean component, built on the MPAS (Model for Prediction Across Scales)
framework using Spherical Centroidal Voronoi Tessellation (SCVT)
unstructured meshes. Each ocean cell is a Voronoi polygon (typically a
hexagon) that can vary in size across the globe. The default E3SM v3 ocean
mesh `IcoswISC30E3r5` is derived from a 30 km icosahedral mesh with ice
shelf cavities (wISC) included around Antarctica.

Notable in-tree externals (under `components/mpas-ocean/src/`):

| External | Purpose |
|----------|---------|
| `MARBL` | Marine Biogeochemistry Library (E3SM's ocean BGC) |
| `BGC` | Older E3SM ocean biogeochemistry |
| `cvmix` | Community Vertical Mixing scheme library |
| `gotm` | General Ocean Turbulence Model |
| `SHTNS` | Spherical harmonic transforms |
| `FFTW` | Fast Fourier transforms |
| `ppr` | Piecewise Parabolic Reconstruction |

Time stepping is split-explicit barotropic / baroclinic. Vertical
coordinate is `z*` (z-star, displaced sea surface) in standard
configurations.

MPAS-Ocean docs: `components/mpas-ocean/docs/{user-guide,tech-guide,dev-guide,design_docs}/`.

The next-generation ocean model **Omega** is under development as a C++
rewrite using Kokkos but is not yet a supported component.

---

## MPAS-Seaice

`components/mpas-seaice/`

The sea ice component, also on an MPAS Voronoi mesh, typically the
identical mesh as the ocean for trivial vertex-cell mapping. Ice
concentration, volume, and tracers live at cell centers; velocity at cell
vertices using B-grid discretization (Arakawa & Lamb 1977).

Column physics is provided by **Icepack**, a CICE Consortium library
included as a submodule under `components/mpas-seaice/src/icepack`. Icepack
provides:

- Mushy-layer thermodynamics (Turner et al. 2013)
- Delta-Eddington / SNICAR-AD radiation
- Level-ice and sealvl melt-pond schemes
- Multi-layer prognostic snow with grain-radius evolution
- Optional aerosols and biogeochemistry

The MPAS-Seaice driver also supports a **prescribed-ice mode** for AMIP-
style F-compset runs that need ice surface fluxes but not a fully
prognostic ice cover.

MPAS-Seaice docs: `components/mpas-seaice/docs/{user-guide,tech-guide,dev-guide}/`.

---

## MOSART (river routing)

`components/mosart/`

The river-routing model. Receives surface and subsurface runoff from ELM,
routes it through a global river network, and discharges to the ocean.
Uses a structured 0.5° or 0.25° grid (compatible with the ELM grid).
Can run standalone as `RMOSGPCC @ r05_r05` for testing.

MOSART docs: `components/mosart/docs/{user-guide,tech-guide,dev-guide}/`.

---

## MALI (land ice) and SeaLevelModel

`components/mpas-albany-landice/`

The land-ice component, built on the MPAS framework using the Albany solver
library for the ice-flow stress balance. Couples to a 1D sea-level model
(`SeaLevelModel/`) for global sea-level fingerprint calculations. Active in
"cryo" compsets (`CRYO1850`, etc.) for polar processes simulation
campaigns.

---

## driver-mct vs driver-moab

E3SM ships **two** coupler drivers and the compset/grid combination
selects which one is used.

| Driver | Library | Use case |
|--------|---------|----------|
| `driver-mct/` | MCT (Model Coupling Toolkit, `externals/mct`) | Default in v1/v2/v3, mature, well-tested |
| `driver-moab/` | MOAB (Mesh-Oriented datABase) | Newer, supports more flexible unstructured-to-unstructured remapping; selected by certain newer compsets and resolutions |

For most users this is invisible: `create_newcase` selects the right
driver based on the compset and resolution. If you are debugging a
remapping issue, check `xmlquery COMP_INTERFACE` to confirm which driver
is active.

---

## CIME: the case control system

`cime/`

CIME is shared with CESM (the NCAR community model), maintained at
https://github.com/ESMCI/cime, and pulled in as a submodule. It provides:

- `create_newcase` (build a CASEROOT directory)
- `case.setup`, `case.build`, `case.submit`
- `xmlchange`, `xmlquery` (modify and read case settings)
- `query_config` (list compsets, grids, machines)
- `create_test`, `wait_for_tests` (regression and integration tests)
- the `pyCIME` Python library underneath all of the above

E3SM-specific extensions live in `cime_config/`:

- `allactive/config_compsets.xml`, coupled compsets (WCYCL*, CRYO*, ...)
- `machines/config_machines.xml`, supported DOE machines
- `machines/config_batch.xml`, batch directives per machine
- `machines/cmake_macros/`, per-machine CMake flags
- `config_grids.xml`, grid aliases
- `tests.py`, `e3sm_developer`, `e3sm_integration`, etc.
- `usermods_dirs/`, reusable case modifications

Per-component CIME glue (default namelists, build scripts, component
compsets) lives under `components/<comp>/cime_config/`.

---

## External submodules summary

Under `externals/`:

| Submodule | Purpose |
|-----------|---------|
| `ekat` | E3SM Kokkos Application Toolkit (foundational C++ utilities for EAMxx) |
| `scorpio` | Parallel I/O library (modern PIO replacement) |
| `scorpio_classic` | Older scorpio interface, kept for compatibility |
| `mam4xx` | C++ port of MAM4 aerosol scheme (used by EAMxx) |
| `mct` | Model Coupling Toolkit (used by `driver-mct`) |
| `YAKL` | Yet Another Kernel Language (used by RRTMGP and some EAMxx code) |
| `cub` | NVIDIA CUB primitives library |
| `ninja` | Ninja build system (used to accelerate CMake builds on some systems) |

---

## Where to next

- Pick a compset and resolution and run a case: `running-cases.md`
- Dig into the atmosphere physics: `eam-atmosphere.md`
- Dig into the ocean and sea ice: `mpas-ocean-seaice.md`
- Inspect what comes out: `output-and-postprocess.md`
- Tune for your machine: `performance-and-scaling.md`
