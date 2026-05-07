# EAM and EAMxx: The Atmosphere Component

E3SM ships **two** atmosphere models. Both are coupled into the same
seven-slot driver, but the source trees, language, and physics are largely
disjoint.

| Model | Language | Status | Compset suffix |
|-------|----------|--------|----------------|
| EAM (EAMv1, v2, v3) | Fortran | Production through E3SM v3 | none (default) |
| EAMxx (SCREAM) | C++ / Kokkos | Next-generation, currently km-scale only | `-SCREAMv1` |

Choose one by picking the appropriate compset. `F1850` and `F2010` use
EAM. `F2010-SCREAMv1` uses EAMxx.

---

## EAMv3: production atmosphere

`components/eam/`

EAMv3 runs on a cubed-sphere mesh (`ne4`, `ne11`, `ne30`, `ne120`,
`ne256`) with `pg2` physics grid. Nominal v3 resolution is `ne30pg2`
(~100 km) with a 1800 s physics time step and 80 vertical layers reaching
~60 km.

### EAMv3 physics suite

The physics package, called the "physics suite", is documented in
`components/eam/docs/tech-guide/`. Each scheme has its own technical guide
page.

| Scheme | What it does | Tech-guide file |
|--------|--------------|-----------------|
| HOMME | Spectral-element dynamical core (transport, dynamics, hyperviscosity) | `homme.md` |
| Atmosphere grid | ne30 cubed-sphere + pg2 physics-grid coupling | `atmosphere-grid-overview.md` |
| ZM (Zhang-McFarlane) | Deep convection: dCAPE-ULL trigger, dilute CAPE closure, entraining plume updraft + downdraft, convective microphysics, mass flux adjustment, MCSP for organized mesoscale convection | `zm.md` |
| P3 | Stratiform cloud microphysics: two-moment, single ice category with predicted rime mass and rime volume, aerosol-dependent ice nucleation | `p3.md` |
| CLUBB | Subgrid turbulence and shallow clouds (PDF-based unified parameterization) | `clubb.md` |
| RRTMG | Radiation: rapid radiative transfer for GCMs, shortwave + longwave | `rrtmg.md` |
| MAM4 / MAM5 | Modal aerosol model (4 or 5 modes; MAM5 adds stratospheric sulfate) | `mam.md` |
| VBS | Volatility basis set for secondary organic aerosols | `vbs.md` |
| Dust | Dust emission parameterization | `dust.md` |
| OCEANFILMS | Sea spray organic aerosol emissions | `oceanfilms.md` |
| chemUCI + Linoz v3 | Tropospheric (chemUCI) and stratospheric (Linoz v3) chemistry | `chemUCIlinozv3.md` |
| ORODRAG | Orographic gravity wave drag | `orodrag.md` |
| COSP | Satellite simulator (ISCCP, MODIS, MISR, CALIPSO, CloudSat, ARM) for diagnostics | `cosp.md` |

### ZM in detail

The Zhang-McFarlane scheme is bulk mass-flux with three pieces:

1. **Trigger**, replaced in E3SMv2 from CAPE-based to **dCAPE-ULL**: dynamic
   CAPE generation (Xie and Zhang 2000) combined with an unrestricted
   parcel launch level (Wang et al. 2015). Reduces overactive afternoon
   convection over land and improves nocturnal mid-level convection.
2. **Cloud model**, entraining plume updraft and downdraft, both saturated.
   Convective microphysics from Song and Zhang (2011) explicitly tracks
   mass and number for cloud water, cloud ice, rain, snow, graupel.
3. **Closure**, dilute CAPE (Neale et al. 2008) sets cloud-base mass flux,
   then the **MAdj** mass flux adjustment couples convection to large-scale
   ascent at PBL top (Song et al. 2023).

**MCSP** (Multiscale Coherent Structure Parameterization) optionally
applies a sinusoidal baroclinic profile to T, q, and momentum to represent
organized mesoscale convective systems.

Namelist switches in `user_nl_eam` for ZM are documented in
`components/eam/docs/user-guide/namelist_parameters.md` under "Zhang and
McFarlane deep convection scheme".

### P3 in detail

P3 (Predicted Particle Properties; Morrison & Milbrandt 2015) replaces the
older microphysics with a single ice category and predicted rime mass and
rime volume mixing ratios. This lets the rime fraction and ice particle
density evolve continuously rather than being binned. Two-moment for cloud
water and ice. Aerosol-dependent ice nucleation: classical nucleation
theory (Liu et al. 2012) for heterogeneous ice nucleation, Liu and Penner
(2005) for homogeneous in-situ cirrus.

### CLUBB

A unified parameterization of subgrid turbulence, shallow clouds, and
boundary layer dynamics using a PDF-based assumed-shape closure for joint
distributions of vertical velocity, temperature, and moisture.

### Radiation: RRTMG

E3SMv3 uses RRTMG (`components/eam/src/physics/rrtmg/`). EAMxx uses the
newer **RRTMGP** (`components/eam/src/physics/rrtmgp/external` submodule
points to E3SM-Project/rte-rrtmgp), which is k-distribution-based and
designed for GPU performance portability through YAKL.

### Aerosols: MAM4 / MAM5

Modal aerosol model with 4 or 5 lognormal modes (Aitken, accumulation,
coarse, primary carbon, plus optional stratospheric sulfate). Tracks
sulfate, black carbon, primary organic, secondary organic, sea salt,
dust, and marine organic aerosol. Coupled to stratiform clouds through
droplet activation (Abdul-Razzak and Ghan 2000) and ice nucleation, and to
aerosol-radiation interactions through the MAM optics tables.

### Chemistry: chemUCI + Linoz v3

`chemUCI` is the comprehensive tropospheric chemistry package (UCI = UC
Irvine). `Linoz v3` is the stratospheric ozone chemistry, parameterized as
a linearized tendency around climatological mean ozone with a first-order
Taylor expansion in local ozone, temperature, and overhead column. Linoz
v3 input file: `linv3_1849-2101_CMIP6_Hist_10deg_58km_c20231207.nc`.

---

## EAMxx (SCREAM): next-generation atmosphere

`components/eamxx/`

EAMxx is a clean-start C++/Kokkos atmosphere model. It is **not** a port
of EAM. The design goals were:

- **Performance portability** across CPU and all major GPU architectures
  (NVIDIA, AMD, Intel) via Kokkos.
- **Modern software practices**: explicit interfaces, unit tests, CI.
- **Cloud-resolving capability** at km-scale (the SCREAM configuration).

Currently shipped only in the SCREAM configuration (Simple Cloud-Resolving
E3SM Atmosphere Model). A low-resolution EAMxx is in development.

### EAMxx physics suite

| Scheme | Source | Notes |
|--------|--------|-------|
| HOMMEXX | `components/eamxx/src/dynamics/homme/` | C++ port of HOMME spectral-element dycore |
| P3 | `components/eamxx/src/physics/p3/` | Same physics ideas as EAM P3, separate C++ implementation |
| SHOC | `components/eamxx/src/physics/shoc/` | Simplified Higher-Order Closure (replaces CLUBB) |
| RRTMGP | `components/eamxx/src/physics/rrtmgp/` | C++/YAKL radiation |
| MAM4xx | `externals/mam4xx` | C++ port of MAM4 aerosols |
| Nudging | `components/eamxx/src/physics/nudging/` | Optional nudging to reanalysis |

### Standalone EAMxx testing

Unique to EAMxx: you can build and test the atmosphere model **without**
CIME or a coupled driver.

```bash
cd components/eamxx

# Run all tests on a machine (requires a compute node on batch systems)
./scripts/test-all-eamxx -m <MACHINE>

# Restrict to a single build type
./scripts/test-all-eamxx -m <MACHINE> -t dbg     # debug only
./scripts/test-all-eamxx -m <MACHINE> -t sp      # single precision only
./scripts/test-all-eamxx -m <MACHINE> -t fpe     # floating-point exceptions
./scripts/test-all-eamxx -m <MACHINE> -t opt     # optimized

# Preserve current shell environment instead of loading machine modules
./scripts/test-all-eamxx --preserve-env -m <MACHINE>

# Compare against local baselines
./scripts/test-all-eamxx -m <MACHINE> --baseline-dir=LOCAL
```

See `components/eamxx/AGENTS.md` for the full standalone testing workflow.

### EAMxx configurations

`components/eamxx/docs/user/` documents these run modes:

- `eamxx_cases.md`, standard CIME case workflow with EAMxx
- `dp_eamxx.md`, doubly-periodic mode (DP-SCREAM, idealized SCM-like)
- `rrm_eamxx.md`, regionally refined mesh mode
- `nudging.md`, nudging to reanalysis
- `multi-instance-rcs.md`, multi-instance ensembles
- `model_configuration.md`, model configuration overview
- `io_aliases.md`, `io_metadata.md`, output configuration
- `python.md`, `py2eamxx.md`, `eamxx2py.md`, Python integration
- `clean_clear_sky.md`, clean and clear-sky radiation diagnostics

---

## Atmosphere namelist customization

In a CIME case, edit `user_nl_eam` (or `user_nl_eamxx` for SCREAM). All
EAM namelist parameters are documented in
`components/eam/docs/user-guide/namelist_parameters.md`. Common ones:

```fortran
! Toggles
 cosp_lite           = .true.            ! enable COSP lite
 empty_htapes        = .true.            ! start from empty history; use only fincl

! History tape configuration (6 tapes shown)
 nhtfrq              = 0,-24,-6,-3,-1,0
 mfilt               = 1,30,120,240,720,1
 avgflag_pertape     = 'A','A','A','A','I','I'

 fincl1 = 'AODALL','CLDLOW',...          ! variables on h0 (monthly)
 fincl2 = 'PS','PRECT','U200',...        ! variables on h1 (daily)
 ! ... etc

! Chemistry
 history_chemdyg_summary    = .true.
 history_gaschmbudget_2D    = .false.
 history_gaschmbudget_2D_levels = .false.

! Aerosols
 is_output_interactive_volc = .true.     ! interactive volcanic SO2 output
```

`nhtfrq` values:
- `0` = monthly average
- positive = output every N model time steps
- negative = output every -N hours (e.g., `-24` = every 24 hours = daily)

`avgflag_pertape` values:
- `'A'` = time-averaged
- `'I'` = instantaneous
- `'M'` = time-minimum
- `'X'` = time-maximum

`mfilt` is the number of time samples written to a single history file
before a new file is opened.

See `output-and-postprocess.md` for the full output convention.

---

## Compset and grid pairing

| Compset | Atmosphere | Default grid |
|---------|------------|--------------|
| `F1850` | EAMv3 | `ne30pg2_r05_IcoswISC30E3r5` |
| `F2010` | EAMv3 | `ne30pg2_r05_IcoswISC30E3r5` |
| `F20TR` | EAMv3 | `ne30pg2_r05_IcoswISC30E3r5` |
| `F2010-SCREAMv1` | EAMxx | finer atmosphere mesh |
| `WCYCL*` | EAMv3 (default) | varies |
| `WCYCL*-MMF1` | EAM with multi-scale modeling framework | varies |

Inputdata for all atmosphere compsets is described in
`components/eam/docs/user-guide/index.md`. SST, sea ice, GHG, solar,
ozone, aerosol emission, and topography files are listed there with
default paths under `inputdata/atm/cam/`.

---

## Where to next

- Land, ocean, ice details: `mpas-ocean-seaice.md`
- Output streams and post-processing: `output-and-postprocess.md`
- GPU performance with EAMxx: `performance-and-scaling.md`
- Submitting an atmosphere PR: `contributing-pr.md`
