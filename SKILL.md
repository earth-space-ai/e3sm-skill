---
name: e3sm
description: >
  Self-contained guide to E3SM, the Energy Exascale Earth System Model
  developed by the U.S. Department of Energy. Covers the coupled atmosphere
  (EAM, EAMxx/SCREAM), land (ELM), ocean (MPAS-Ocean), sea ice (MPAS-Seaice),
  river (MOSART), and land ice (MALI) components, plus the CIME case-control
  workflow, supported DOE machines, output diagnostics, performance tuning,
  debugging, and the GitHub-based contribution process. Progressive disclosure:
  start in this file for routing, drill into reference/ for depth.
version: 0.1.0
tags:
  - earth-system-model
  - climate
  - coupled-model
  - atmosphere
  - ocean
  - land-surface
  - sea-ice
  - cime
  - hpc
  - kokkos
  - gpu
  - doe
  - e3sm
---

# E3SM (Energy Exascale Earth System Model): Complete Guide

> **E3SM** = Energy Exascale Earth System Model
> Project lead: U.S. Department of Energy (DOE) Office of Biological and Environmental Research
> Source: https://github.com/E3SM-Project/E3SM
> Documentation: https://docs.e3sm.org/E3SM
> DOI: 10.11578/E3SM/dc.20240930.1
> v2 description paper: Golaz et al. 2022, JAMES, doi:10.1029/2022MS003156

**What E3SM does:** Solves the coupled equations of a fully active Earth
system (atmosphere, ocean, land, sea ice, river routing, land ice) on
unstructured spherical-element and Voronoi meshes, at resolutions from a
nominal 100 km coupled grid (`ne30pg2_r05_IcoswISC30E3r5`) to km-scale
cloud-resolving SCREAM. Designed for DOE Leadership Computing Facilities
(Frontier, Perlmutter, Aurora) with hybrid CPU+GPU performance portability
through Kokkos.

**Who this skill is for:** Agents, postdocs, and computational scientists
who need to clone, build, configure, run, analyze, debug, and contribute to
the E3SM coupled model on a supported HPC system.

---

## Quick Decision Tree

```
"What do I need?"
│
├─ First time. What is E3SM and how do I get the code on a supported machine?
│  └─ Read: reference/getting-started.md
│     (clone with git-fleximod externals, supported DOE machines, CIME)
│
├─ I want to understand which components live where in the source tree
│  └─ Read: reference/architecture.md
│     (EAM, EAMxx, ELM, MPAS-Ocean, MPAS-Seaice, MOSART, driver-mct vs driver-moab)
│
├─ I want to run a coupled or single-component case
│  └─ Read: reference/running-cases.md
│     (create_newcase, compsets WCYCL/F/I, ne30/ne120/ne256, build, submit)
│
├─ I am working on the atmosphere physics (ZM, P3, CLUBB, RRTMGP, MAM)
│  └─ Read: reference/eam-atmosphere.md
│     (EAM physics suite vs EAMxx/SCREAM, namelists, COSP)
│
├─ I am working on the ocean or sea ice
│  └─ Read: reference/mpas-ocean-seaice.md
│     (unstructured Voronoi mesh, time stepping, MARBL BGC, Icepack column physics)
│
├─ I want to inspect or post-process output
│  └─ Read: reference/output-and-postprocess.md
│     (h0/h1/h2/h3 history streams, e3sm_diags, mpas_analysis, zppy)
│
├─ I need to tune PE layout, threading, or run on GPU
│  └─ Read: reference/performance-and-scaling.md
│     (PELAYOUT, threading, Kokkos GPU, Frontier/Perlmutter notes)
│
├─ My case crashed at build or run time, or restart is not bit-for-bit
│  └─ Read: reference/debugging.md
│     (build failures, e3sm.log/cesm.log, conservation, BFB testing)
│
└─ I want to submit a pull request to E3SM-Project/E3SM
   └─ Read: reference/contributing-pr.md
      (fork, feature branch naming, integration tests, NextGen vs maintenance)
```

---

## What's Inside E3SM

```
E3SM/
├── README.md · AGENTS.md · CONTRIBUTING.md · CITATION.cff · LICENSE
├── mkdocs.yaml                          ← top-level MkDocs config (docs.e3sm.org)
├── run_e3sm.template.sh                 ← example coupled run script
│
├── cime/                                ← CIME submodule (case control system)
├── cime_config/                         ← E3SM-specific CIME configuration
│   ├── allactive/config_compsets.xml    ← coupled compsets (WCYCL*, CRYO*)
│   ├── machines/config_machines.xml     ← supported DOE machines
│   ├── machines/config_batch.xml        ← batch system configs
│   ├── machines/cmake_macros/           ← per-machine build flags
│   ├── config_grids.xml                 ← grid aliases (ne30pg2_r05_*, ...)
│   ├── tests.py                         ← e3sm_*_developer test suites
│   └── usermods_dirs/                   ← reusable case modifications
│
├── components/
│   ├── eam/                             ← E3SM Atmosphere Model (Fortran, EAMv3)
│   │   ├── src/{dynamics,physics,...}
│   │   ├── docs/{tech-guide,user-guide,dev-guide}
│   │   └── cime_config/
│   ├── eamxx/                           ← Next-gen atmosphere (C++/Kokkos, SCREAM)
│   │   ├── src/{dynamics,physics,share,...}
│   │   ├── scripts/test-all-eamxx
│   │   └── AGENTS.md                    ← standalone testing guide
│   ├── elm/                             ← E3SM Land Model (Fortran, fork of CLM)
│   │   └── src/external_models/{fates,mpp,sbetr}
│   ├── mosart/                          ← River routing
│   ├── mpas-ocean/                      ← Ocean (unstructured Voronoi)
│   │   └── src/{MARBL, cvmix, BGC, gotm, SHTNS, FFTW}
│   ├── mpas-seaice/                     ← Sea ice (MPAS + Icepack column physics)
│   │   └── src/icepack
│   ├── mpas-albany-landice/             ← MALI land-ice / SeaLevelModel
│   ├── mpas-framework/                  ← Shared MPAS infrastructure
│   ├── homme/                           ← Spectral element dycore (used by EAM)
│   ├── cice/                            ← Legacy CICE (rarely used in modern compsets)
│   ├── ww3/                             ← WaveWatch III (optional)
│   ├── data_comps/                      ← Data atmosphere/ocean/ice (D{ATM,OCN,ICE,...})
│   ├── stub_comps/                      ← Stub components (S{ATM,LND,OCN,...})
│   └── xcpl_comps/                      ← Test exchange components
│
├── driver-mct/                          ← MCT-based coupler driver (default)
├── driver-moab/                         ← MOAB-based coupler driver (newer)
├── share/                               ← Shared utilities (timing, RNG, streams)
├── externals/                           ← Submodules: ekat, scorpio, mam4xx,
│                                          haero, YAKL, mct, cub, ninja
├── docs/                                ← MkDocs source for docs.e3sm.org
└── tools/                               ← Diagnostic and grid generation tools
```

**For most users:** edit `user_nl_*` files and `xmlchange` settings inside a
`CASEROOT` created by `create_newcase`. Source code edits live under
`components/<comp>/src/`. You almost never edit anything inside `cime/` or
`externals/` directly, those are submodules pinned to upstream releases.

---

## Critical Rules

1. **Never run `create_newcase` outside a supported machine.** CIME selects
   compilers, batch directives, and module environments from
   `cime_config/machines/config_machines.xml`. On an unsupported host, the
   case will be created but `case.build` will fail with cryptic missing-module
   errors. Run `./query_config --machines` from `cime/scripts` to confirm.

2. **Build is out-of-source.** The case directory (`CASEROOT`), build
   directory (`EXEROOT`), and run directory (`RUNDIR`) are three distinct
   filesystems. Never modify `EXEROOT`/`RUNDIR` by hand. Use `xmlchange` and
   `xmlquery` to inspect and adjust them.

3. **Initialize submodules with `--recursive --depth=1`.** E3SM tracks
   ~20 submodules (CIME, MCT, scorpio, EKAT, YAKL, mam4xx, MARBL, Icepack,
   FATES, ...). A clone without `git submodule update --init --recursive`
   produces empty subdirectories and immediate build failures. The
   recommended invocation:

   ```bash
   git clone --recursive https://github.com/E3SM-Project/E3SM.git
   # or, after a non-recursive clone:
   git submodule update --init --recursive --depth=1
   ```

4. **Modifying `pelayout` requires `case.setup --reset`.** Changing
   `NTASKS`, `NTHRDS`, or `ROOTPE` in the case directory will not take
   effect until you re-run `case.setup --reset` followed by `case.build`.
   Just rebuilding is not enough.

5. **EAMxx and EAM are different code paths.** `F1850` and `F2010` use EAM
   (Fortran). `F2010-SCREAMv1` uses EAMxx (C++/Kokkos). Editing
   `components/eam/src/` does not affect SCREAM cases, and vice versa.

6. **Coupled testing is expensive. Use scoped compsets.**
   - Touched only ELM physics: `I1850CNPRDCTCBCTOP @ ne4pg2_ne4pg2`
   - Touched only EAM: `F1850 @ ne4pg2_oQU480`
   - Touched only EAMxx: `F2010-SCREAMv1 @ ne4pg2_ne4pg2`
   - Touched MOSART only: `RMOSGPCC @ r05_r05`
   - Touched MPAS-Ocean: `CMPASO-NYF @ T62_oQU120`
   - Touched 2+ components: coupled `WCYCL1850NS @ ne4pg2_r05_oQU480`

7. **PR feature branch naming follows `<github_user>/<area>/<topic>`.** The
   E3SM project enforces this through CI; PRs with arbitrary branch names
   are flagged. Example: `kwu/eam/zm-trigger-fix`.

---

## Quick Start (coupled WCYCL1850 on Perlmutter)

```bash
# 1. Get the code (recursive picks up CIME, MCT, scorpio, MARBL, Icepack, ...)
git clone --recursive https://github.com/E3SM-Project/E3SM.git
cd E3SM

# 2. Create a case (replace MACHINE; query_config --machines lists supported)
cd cime/scripts
./create_newcase \
    --case ../../cases/test_wcycl \
    --compset WCYCL1850 \
    --res ne30pg2_r05_IcoswISC30E3r5 \
    --mach pm-cpu \
    --pecount M \
    --project <your_DOE_allocation>

# 3. Setup, build, submit
cd ../../cases/test_wcycl
./case.setup
./case.build                   # ~30-90 min depending on machine and compset
./xmlchange STOP_OPTION=ndays,STOP_N=5
./case.submit

# 4. Inspect output
./xmlquery RUNDIR              # path to history files
ls $(./xmlquery --value RUNDIR)/*.eam.h0.*.nc
```

For SCREAM (km-scale atmosphere) replace `--compset WCYCL1850` with
`--compset F2010-SCREAMv1` and use a finer atmosphere mesh.

For a more complete production-style script see `run_e3sm.template.sh` in
the repo root, which wraps `create_newcase`, custom PE layout, namelist
edits, and `case.submit` into a single bash entry point.

→ Full walkthrough in `reference/getting-started.md` and
`reference/running-cases.md`.

---

## Reference Documents

| Document | What's inside |
|----------|---------------|
| `reference/getting-started.md` | Clone with externals, CIME submodule, supported DOE machines, build prerequisites, quickstart on a new login node |
| `reference/architecture.md` | Coupled-model layout, EAM vs EAMxx, ELM vs CLM/CTSM heritage, MPAS-Ocean / MPAS-Seaice, MOSART, MALI, driver-mct vs driver-moab |
| `reference/running-cases.md` | `create_newcase`, compset families (WCYCL/F/I/G/B/D), supported resolutions (ne4/ne11/ne30/ne120/ne256/RRM), case workflow (setup → build → submit), `xmlchange`, `user_nl_*`, restarts |
| `reference/eam-atmosphere.md` | EAMv3 physics suite (HOMME, ZM, P3, CLUBB, RRTMG, MAM, COSP, chemUCI/Linoz), EAMxx/SCREAM (C++/Kokkos, P3, SHOC, RRTMGP), namelist parameters |
| `reference/mpas-ocean-seaice.md` | Voronoi unstructured mesh, split-explicit time stepping, MARBL biogeochemistry, Icepack column physics, ice shelf cavities (wISC), prescribed-ice mode |
| `reference/output-and-postprocess.md` | History tape conventions (h0 monthly, h1 daily, h2 6-hourly, h3 hourly, h4 instantaneous), `nhtfrq`/`mfilt`/`fincl`/`fexcl`, e3sm_diags, mpas_analysis, zppy |
| `reference/performance-and-scaling.md` | PE layouts (XS/S/M/L/custom-N), threading via `NTHRDS`, GPU support via Kokkos/YAKL, machine-specific notes (Frontier, Perlmutter, Chrysalis, Aurora), scorpio I/O |
| `reference/debugging.md` | Build failures (CMake, missing deps), runtime crashes, `e3sm.log.*` and `cesm.log.*`, conservation diagnostics, restart bit-for-bit, BFB testing with PEM/PET/ERS |
| `reference/contributing-pr.md` | Fork workflow, feature branch naming, integration test suites (`e3sm_developer`, `e3sm_integration`), NextGen branch vs `master`, EAMxx PR considerations, CI on `my.cdash.org` |
