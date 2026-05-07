# Running E3SM Cases

E3SM is configured and launched through CIME. The unit of work is a
**case**: a directory created by `create_newcase` that holds the choice of
compset, grid, machine, PE layout, run-time settings, and namelist
modifications. This page walks through the case lifecycle and the most
common knobs you will turn.

---

## The case lifecycle

```
1. create_newcase   →  CASEROOT directory exists, XML config populated
2. case.setup       →  namelists generated, env_*.xml locked, build dir prepared
3. case.build       →  e3sm.exe built in EXEROOT
4. case.submit      →  job queued through batch system; output lands in RUNDIR
5. case.submit      →  again, with CONTINUE_RUN=TRUE, for restart segments
```

Three filesystem locations matter:

| Variable | What it holds | How to query |
|----------|---------------|--------------|
| `CASEROOT` | XML config, namelists, run scripts (you edit here) | `pwd` from inside the case |
| `EXEROOT` | Build artifacts (`*.o`, `*.a`, `e3sm.exe`) | `./xmlquery EXEROOT` |
| `RUNDIR` | History files, log files, restart files (model writes here) | `./xmlquery RUNDIR` |

You should never edit anything inside `EXEROOT` or `RUNDIR` directly. Use
`xmlchange` to modify settings; CIME will regenerate downstream files.

---

## create_newcase

```bash
cd cime/scripts
./create_newcase \
    --case <PATH/TO/CASEROOT> \
    --compset <COMPSET_ALIAS> \
    --res <GRID_ALIAS> \
    --mach <MACHINE> \
    --project <DOE_ALLOCATION> \
    [--pecount {XS,S,M,L,X1,X2,custom-N}] \
    [--walltime HH:MM:SS] \
    [--case-group GROUP_NAME] \
    [--script-root PATH] \
    [--output-root PATH] \
    [--handle-preexisting-dirs {a,r,u}]
```

`--handle-preexisting-dirs u` is "use" (skip and reuse), helpful when
re-running a setup script.

After this completes, `CASEROOT` contains:

```
CaseDocs/                    ← documented namelists for every component
SourceMods/                  ← drop in modified source files here (no rebuild needed for some)
Tools/                       ← CIME case utilities
case.build · case.setup · case.submit · case.st_archive · ...
xmlchange · xmlquery
preview_namelists · preview_run
env_archive.xml · env_batch.xml · env_build.xml · env_case.xml · env_mach_pes.xml
env_mach_specific.xml · env_run.xml · env_workflow.xml
README.case                  ← human-readable summary
user_nl_eam · user_nl_elm · user_nl_mosart · user_nl_cpl · ...
```

`README.case` records the longname of the compset (the fully expanded
form with explicit physics suffixes), useful for reporting bugs.

---

## Compsets

A **compset** ("component set") names the seven-component configuration of
a simulation. The longname format is:

```
TIME_ATM[%phys]_LND[%phys]_ICE[%phys]_OCN[%phys]_ROF[%phys]_GLC[%phys]_WAV[%phys]
```

For convenience, short **aliases** exist. The full alias-to-longname mapping
lives in `cime_config/allactive/config_compsets.xml` (coupled) and per
component at `components/*/cime_config/config_compsets.xml`.

```bash
cd cime/scripts
./query_config --compsets all       # all known compsets across all components
./query_config --compsets eam       # just EAM-active compsets
```

### Coupled "WCYCL" family: water cycle simulation campaign

Standard fully coupled configurations (active EAM, ELM, MPASO, MPASSI,
MOSART, MALI/SGLC, WW3/SWAV).

| Alias | Forcing |
|-------|---------|
| `WCYCL1850` | Pre-industrial climatological (1850) forcings |
| `WCYCL1850-4xCO2` | 1850 + abrupt 4xCO2 |
| `WCYCL1850-1pctCO2` | 1850 + 1% per year CO2 increase |
| `WCYCL1950` | Perpetual 1950 forcings |
| `WCYCL20TR` | Historical transient (1850-2014) |
| `WCYCLSSP245` / `SSP370` / `SSP585` | Future scenarios |
| `WCYCL1850NS` | 1850 with no spun-up ICs (testing only) |
| `WCYCL2010NS` | 2010 with no spun-up ICs (testing only) |
| `WCYCL20TR-GHG` / `-aer` / `-nat` / ... | Single-forcing historical experiments |

The `-CMIP7` suffix selects the CMIP7 forcing variant. The `-MMF1`
suffix selects the Multi-Scale Modeling Framework superparameterization.

### Atmosphere-only "F" family

Active EAM (or EAMxx with `-SCREAMv1`), data ocean (DOCN), data ice
(DICE), active ELM, active MOSART. Ocean SSTs and sea ice are prescribed
from input data.

| Alias | Forcing |
|-------|---------|
| `F1850` | Pre-industrial atmosphere only |
| `F2010` | Year-2010 climatology |
| `F20TR` | Historical 1850-2014 with time-varying SSTs and emissions |
| `F2010-SCREAMv1` | EAMxx (SCREAM) at 2010 conditions |

### Land-only "I" family

Active ELM with DATM (data atmosphere reading observed forcing), stub
ocean and ice. Used for ELM physics development.

| Alias | Forcing |
|-------|---------|
| `I1850CNPRDCTCBCTOP` | Pre-industrial CNP-cycle with prognostic top |
| `I20TRCNPRDCTCBCTOP` | Historical CNP cycle |
| `IELMBC` | ELM bypass coupler mode (very fast for testing) |

### Ocean-only "G" / "C" family

Active MPAS-Ocean with DATM. Used for ocean physics development.

| Alias | Forcing |
|-------|---------|
| `CMPASO-NYF` | MPAS-Ocean with normal-year forcing |

### Sea-ice-only "D" family

Active MPAS-Seaice with prescribed atmosphere and ocean.

| Alias | Forcing |
|-------|---------|
| `DTESTM` | Sea ice testing compset |

### River-only "RMOS" family

| Alias | Forcing |
|-------|---------|
| `RMOSGPCC` | MOSART with GPCC runoff forcing |

### Picking a compset for a focused PR

If your code touches **only one** component, use the matching scoped
compset. From `AGENTS.md`:

| Touched | Compset | Resolution |
|---------|---------|------------|
| ELM | `I1850CNPRDCTCBCTOP` | `ne4pg2_ne4pg2` |
| EAM | `F1850` | `ne4pg2_oQU480` |
| EAMxx | `F2010-SCREAMv1` | `ne4pg2_ne4pg2` |
| MOSART | `RMOSGPCC` | `r05_r05` |
| MPAS-Ocean | `CMPASO-NYF` | `T62_oQU120` |
| MPAS-Seaice | `DTESTM` | `T62_oQU240` |
| EAM and ELM together | `F1850` | `ne4pg2_oQU480` |
| 2+ components | `WCYCL1850NS` (coupled) | `ne4pg2_r05_oQU480` |

These low-resolution combinations are not scientifically meaningful but
are designed to build and run in minutes.

---

## Resolutions (grids)

Grids in CIME have an alias of the form
`a%name_l%name_oi%name_r%name_m%mask_g%name_w%name`, but most users use
short combination aliases. The full mapping is in
`cime_config/config_grids.xml`.

```bash
cd cime/scripts
./query_config --grids                    # all grids
./query_config --grids alias=ne30pg2_r05_IcoswISC30E3r5
```

### Common combinations

| Alias | Atmosphere | Land/River | Ocean/Ice |
|-------|------------|------------|-----------|
| `ne4pg2_oQU480` | ne4 cubed-sphere (~750 km) | r05 (0.5°) | QU 480 km |
| `ne4pg2_r05_oQU480` | ne4 (~750 km) | r05 | QU 480 km (coupled testing) |
| `ne4pg2_ne4pg2` | ne4 (~750 km) | ne4 (atm grid) | (data ocean) |
| `ne11_oQU240` | ne11 (~340 km) | r05 | QU 240 km |
| `ne30pg2_r05_IcoswISC30E3r5` | ne30 (~100 km) | r05 | Icosahedral 30 km, ISC | (E3SMv3 standard low-res) |
| `northamericax4v1pg2_r025_IcoswISC30E3r5` | NA-RRM (110 → 25 km) | r025 | Icos 30 km | (E3SMv3 RRM) |
| `ne120pg2_r025_IcoswISC30E3r5` | ne120 (~25 km) | r025 | Icos 30 km | (high-res) |
| `ne256pg2_r025_oRRS18to6` | ne256 (~12 km) | r025 | RRS 18-to-6 km | (very high-res, expensive) |
| `T62_oQU120` | T62 data atm | (none) | QU 120 km |

### Atmosphere mesh notation

- `ne30` = cubed-sphere with 30 spectral elements per cube edge → ~100 km
  nominal at the equator. `ne4` is ~750 km. `ne120` is ~25 km. `ne256` is
  ~12 km.
- `pg2` = "physics grid 2", a 2x2 finite-volume sub-cell decomposition
  of each spectral element used for tracer transport and parameterized
  physics. Standard since v2.
- RRM (regionally refined mesh) names like `northamericax4v1` indicate a
  factor-of-4 refinement over the named region.

### Ocean mesh notation

- `oQU480`, `oQU240`, `oQU120` = quasi-uniform Voronoi at 480/240/120 km.
- `Icos30E3r5` = icosahedral 30 km, E3SMv3 revision r5.
- `wISC` suffix = with ice shelf cavities (used for ice-sheet coupling).
- `oRRS18to6` = regionally refined 18 km background, 6 km in target region.

### SCREAM resolutions

SCREAM is designed for cloud-resolving km-scale simulations. Production
SCREAM cases use `ne1024pg2` (~3 km) and similar. These are extreme
configurations that require exascale-class machines and full system
allocations.

---

## case.setup

```bash
./case.setup                      # generate namelists, lock env_case
./case.setup --reset              # recompute namelists from scratch (after PE layout changes)
```

Use `./case.setup --reset` whenever you change `NTASKS`, `NTHRDS`, or
`ROOTPE`. Plain `case.setup` will not pick up PE layout changes.

After this, namelists have been generated in `RUNDIR` and a copy is in
`CaseDocs/` for reference. Run `./preview_namelists` to regenerate the
documented copies after editing `user_nl_*` files.

---

## case.build

```bash
./case.build                      # full build, picks up source mods from SourceMods/
./case.build -m                   # only rebuild driver and components, skip externals
./case.build --clean              # nuke and rebuild
```

Build time:

| Compset | Machine | Time |
|---------|---------|------|
| `WCYCL1850NS` ne4pg2 | Perlmutter login | 15-30 min |
| `WCYCL1850` ne30pg2 | Perlmutter compute | 30-60 min |
| `F2010-SCREAMv1` | Perlmutter GPU | 30-90 min |
| anything | first build, cold cache | add 10-20 min |

If you only changed component source code (not the driver or externals),
`./case.build -m` is significantly faster.

If you changed PE layout, use:

```bash
./case.setup --reset
./case.build
```

---

## case.submit

```bash
./xmlchange STOP_OPTION=ndays,STOP_N=5
./xmlchange RESUBMIT=0
./case.submit
```

Common STOP_OPTION values: `nsteps`, `nseconds`, `nminutes`, `nhours`,
`ndays`, `nmonths`, `nyears`, `date`. STOP_N is the count.

For long simulations, set `RESUBMIT=N` so the system automatically queues
N more segments after each one finishes (subject to walltime per
segment).

Restart frequency:

```bash
./xmlchange REST_OPTION=nyears,REST_N=5
```

This writes a restart bundle every 5 simulated years.

To continue a previously completed run:

```bash
./xmlchange CONTINUE_RUN=TRUE
./case.submit
```

The model will pick up from the most recent `rpointer.*` files in
`RUNDIR`.

---

## Customizing namelists

Each component reads its namelist from `user_nl_<comp>` in `CASEROOT`.
Settings here override the defaults from `components/<comp>/cime_config/`.

Example `user_nl_eam` snippet (taken from `run_e3sm.template.sh`):

```fortran
 cosp_lite = .true.
 empty_htapes = .true.

 avgflag_pertape = 'A','A','A','A','I','I'
 nhtfrq = 0,-24,-6,-3,-1,0
 mfilt  = 1,30,120,240,720,1

 fincl1 = 'AODALL','AODBC','CLDLOW','CLDMED','CLDHGH','CLDTOT', ...
 fincl2 = 'PS','FLUT','PRECT','U200','V200', ...
```

This says: 6 history tapes, monthly h0 (averaged), daily h1 (averaged), 6-
hourly h2, 3-hourly h3, hourly h4 (instantaneous), and one h5 file
(instantaneous, 1 file). See `output-and-postprocess.md` for the full
convention.

Always run `./preview_namelists` after editing `user_nl_*` so the
`CaseDocs/` copies are updated.

---

## xmlchange and xmlquery

Modify CIME-tracked settings:

```bash
./xmlchange STOP_OPTION=nyears,STOP_N=10
./xmlchange --append --id CAM_CONFIG_OPTS --val='-cosp'
./xmlchange NTASKS_ATM=512,NTASKS_LND=128,NTASKS_OCN=256
./xmlchange DEBUG=TRUE                       # turn on debug build
./xmlchange BUILD_COMPLETE=TRUE              # mark a build as complete (skip rebuild)
```

Read settings:

```bash
./xmlquery STOP_OPTION,STOP_N
./xmlquery --value RUNDIR                    # raw value, no formatting
./xmlquery COMP_ATM,COMP_LND,COMP_OCN        # which models in each slot
./xmlquery PIO_TYPENAME                      # I/O backend
./xmlquery --partial NTASKS                  # everything containing NTASKS
```

The `env_*.xml` files in CASEROOT can also be edited by hand, but
`xmlchange` is preferred because it validates the change and updates any
dependent settings.

---

## A complete production-style run script

For long production simulations the E3SM team uses a single bash wrapper
that drives `create_newcase`, custom PE layout, namelist edits, build,
and submit. See `run_e3sm.template.sh` in the E3SM repo root. Key
sections to customize:

| Variable | Meaning |
|----------|---------|
| `MACHINE` | One of `pm-cpu`, `frontier`, `chrysalis`, ... |
| `COMPSET` | e.g., `WCYCL1850` |
| `RESOLUTION` | e.g., `ne30pg2_r05_IcoswISC30E3r5` |
| `CASE_NAME` | Your case identifier (gets appended to paths) |
| `CHECKOUT` | Date string like `20240301` (chooses which clone of E3SM) |
| `BRANCH` | Usually `master` |
| `MODEL_START_TYPE` | `initial`, `continue`, `branch`, `hybrid` |
| `START_DATE` | e.g., `0001-01-01` |
| `run` | `XS_2x5_ndays` (test) or `production` |
| `PELAYOUT`, `WALLTIME`, `STOP_OPTION`, `STOP_N`, `RESUBMIT` | Standard CIME settings |

The script also defines `user_nl()` and `patch_mpas_streams()` functions
where you write any namelist customizations.

---

## Where to next

- Atmosphere physics details: `eam-atmosphere.md`
- Ocean and sea ice details: `mpas-ocean-seaice.md`
- Inspect output: `output-and-postprocess.md`
- Tune for your machine and architecture: `performance-and-scaling.md`
- Diagnose a build or run failure: `debugging.md`
