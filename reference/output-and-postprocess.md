# Output and Postprocessing

Each E3SM component writes its own history files into `RUNDIR`, on its own
schedule, in its own format. This page covers the file naming
conventions, how to control what is written, and the standard
postprocessing tools.

---

## File naming conventions

After a run finishes, `RUNDIR` typically contains:

```
<casename>.eam.h0.YYYY-MM.nc                       ← EAM monthly history
<casename>.eam.h1.YYYY-MM-DD-SSSSS.nc              ← EAM daily history
<casename>.eam.h2.YYYY-MM-DD-SSSSS.nc              ← EAM 6-hourly history
<casename>.eam.h3.YYYY-MM-DD-SSSSS.nc              ← EAM 3-hourly history
<casename>.eam.h4.YYYY-MM-DD-SSSSS.nc              ← EAM hourly history
<casename>.eam.r.YYYY-MM-DD-SSSSS.nc               ← EAM restart
<casename>.eam.i.YYYY-MM-DD-SSSSS.nc               ← EAM initial (cold start template)

<casename>.elm.h0.YYYY-MM.nc                       ← ELM monthly
<casename>.elm.h1.YYYY-MM-DD-SSSSS.nc              ← ELM higher-frequency
<casename>.elm.r.YYYY-MM-DD-SSSSS.nc               ← ELM restart

<casename>.mosart.h0.YYYY-MM.nc                    ← MOSART monthly
<casename>.mosart.r.YYYY-MM-DD-SSSSS.nc            ← MOSART restart

<casename>.mpaso.hist.am.timeSeriesStatsMonthly.YYYY-MM-01.nc
<casename>.mpaso.hist.am.timeSeriesStatsDaily.YYYY-MM-DD.nc
<casename>.mpaso.rst.YYYY-MM-DD_00000.nc

<casename>.mpassi.hist.am.timeSeriesStatsMonthly.YYYY-MM-01.nc
<casename>.mpassi.rst.YYYY-MM-DD_00000.nc

<casename>.cpl.hi.YYYY-MM-DD-SSSSS.nc              ← Coupler history (every HIST_N units)
<casename>.cpl.r.YYYY-MM-DD-SSSSS.nc               ← Coupler restart

rpointer.atm · rpointer.lnd · rpointer.ocn · rpointer.ice · ...
                                                   ← restart pointer files
```

For SCREAM cases, `eam` is replaced by `eamxx` (or by `scream` in older
builds).

---

## EAM history tape convention

EAM (and ELM, MOSART) writes a configurable number of **history tapes**,
indexed `h0`, `h1`, `h2`, ... `h9`. Each tape is independently configured
through `nhtfrq`, `mfilt`, `avgflag_pertape`, `fincl<N>`, `fexcl<N>`.

By default, only `h0` is active and contains a standard set of monthly
averaged fields. To customize, edit `user_nl_eam`:

```fortran
 empty_htapes = .true.            ! start from empty defaults; only emit fincl<N> variables

 nhtfrq          = 0,-24,-6,-3,-1,0
 mfilt           = 1,30,120,240,720,1
 avgflag_pertape = 'A','A','A','A','I','I'

 fincl1 = 'PS','TS','PRECT', ... (monthly variables)
 fincl2 = 'PS','PRECT','U200',  ... (daily variables)
 fincl3 = 'PSL','U850','V850', ... (6-hourly variables)
 fincl4 = 'PRECT'                  (3-hourly variable)
 fincl5 = 'O3_SRF'                 (hourly variable, instantaneous)
 fincl6 = 'CO_2DMSD','NO2_2DMSD', ... (instantaneous snapshot)
```

### `nhtfrq` semantics

| Value | Meaning |
|-------|---------|
| `0` | Monthly average (start of month boundary) |
| Positive `N` | Output every N model time steps |
| Negative `-N` | Output every N hours (e.g., `-24` = daily, `-6` = 6-hourly, `-1` = hourly) |

### `mfilt` semantics

Number of time samples per file before opening a new file. With
`mfilt=30` and daily output (`nhtfrq=-24`), each file holds 30 days
of data.

### `avgflag_pertape` values

| Value | Meaning |
|-------|---------|
| `'A'` | Time-averaged over the output interval |
| `'I'` | Instantaneous snapshot at the output time |
| `'M'` | Time-minimum over the output interval |
| `'X'` | Time-maximum over the output interval |

### `fincl` and `fexcl`

`fincl<N>` adds variables to tape N. `fexcl<N>` removes variables from
tape N (most useful when `empty_htapes = .false.` to remove default
variables from h0).

Some variables can be tagged with averaging flags inline:

```fortran
 fincl1 = 'TREFHTMX:X','TREFHTMN:N','PRECT:A'
```

`:X` = max, `:N` = min, `:A` = average, `:I` = instantaneous, regardless
of the tape's default `avgflag_pertape`.

---

## ELM history tape convention

ELM uses the same h0/h1/... tape system through namelist variables
`hist_nhtfrq`, `hist_mfilt`, `hist_avgflag_pertape`, `hist_fincl<N>`,
`hist_fexcl<N>` in `user_nl_elm`. Example:

```fortran
 hist_dov2xy        = .true.,.true.       ! map from PFT/column to grid
 hist_fincl1        = 'SNOWDP','COL_FIRE_CLOSS','NPOOL','PPOOL','TOTPRODC'
 hist_fincl2        = 'H2OSNO','FSNO','QRUNOFF','QSNOMELT', ...
 hist_mfilt         = 1,365
 hist_nhtfrq        = 0,-24
 hist_avgflag_pertape = 'A','A'
 hist_fexcl1        = 'AGWDNPP','ALTMAX_LASTYEAR', ...
```

`hist_dov2xy = .true.` maps the PFT-level state to a gridcell
representation suitable for output.

---

## MPAS-Ocean and MPAS-Seaice streams

MPAS components do not use the h0/h1 tape convention. Instead they
maintain a list of **streams** (like Unix file descriptors), each
configurable in an XML file generated at case setup:

```
<RUNDIR>/streams.ocean
<RUNDIR>/streams.seaice
```

The most common output streams are written by **analysis members**
(modules under `components/mpas-ocean/src/analysis_members/` and
`components/mpas-seaice/src/analysis_members/`):

| Stream | What it contains |
|--------|------------------|
| `timeSeriesStatsMonthly` | Monthly time-mean of selected fields |
| `timeSeriesStatsDaily` | Daily time-mean |
| `timeSeriesStatsCustom` | Custom averaging window |
| `globalStats` | Global integrals (volume, heat, salt, ice extent) |
| `meridionalHeatTransport` | MHT diagnostic |
| `mixedLayerDepths` | MLD computed from a chosen criterion |

Streams are described in greater detail at
http://docs.e3sm.org/polaris/main/ and in `mpas_analysis` documentation.

To enable, disable, or customize a stream:

1. Edit the auto-generated `streams.ocean` (or `streams.seaice`) in
   `RUNDIR` directly, OR
2. Set the corresponding namelist option in `user_nl_mpaso` (or
   `user_nl_mpassi`).

---

## Coupler output

The coupler driver writes its own history (`cpl.hi`) and restart
(`cpl.r`) files, controlled by:

```bash
./xmlchange HIST_OPTION=nyears,HIST_N=5     # write coupler hi every 5 years
./xmlchange BUDGETS=TRUE                    # always on for production
```

Coupler history captures the fluxes exchanged between every pair of
components (atm-ocn, atm-lnd, ocn-ice, ...) and is essential for
diagnosing conservation issues.

---

## Standard postprocessing tools

### e3sm_diags

Atmosphere-focused diagnostics package: model-vs-observation comparisons,
zonal means, polar plots, lat-lon maps, Taylor diagrams, ENSO indices,
ARM diagnostics.

- Repo: https://github.com/E3SM-Project/e3sm_diags
- Docs: https://docs.e3sm.org/e3sm_diags/
- Install via conda: `conda install -c conda-forge e3sm_diags`

Typical workflow:

```bash
e3sm_diags_driver.py -p run.cfg
```

where `run.cfg` points to your monthly `eam.h0.*.nc` files and a set of
observational reference data.

### mpas_analysis

Ocean and sea ice diagnostics for MPAS components: SST, SSS, sea ice
extent and thickness, AMOC, MHT, mixed-layer depth, regional climatology
plots.

- Repo: https://github.com/MPAS-Dev/MPAS-Analysis
- Docs: https://mpas-dev.github.io/MPAS-Analysis/latest/
- Install via conda: `conda install -c conda-forge mpas-analysis`

Typical workflow:

```bash
mpas_analysis run.cfg
```

### zppy (E3SM unified post-processing)

End-to-end pipeline that orchestrates climatology generation, time
series extraction, e3sm_diags, mpas_analysis, ILAMB, and global time
series plots into a single SLURM-managed workflow.

- Repo: https://github.com/E3SM-Project/zppy
- Docs: https://e3sm-project.github.io/zppy/

Configured via a single `.cfg` file with sections for each task type.
zppy is the recommended entry point for complete diagnostic packages on
production runs.

### ILAMB (land model benchmarking)

Land model benchmarking against an observational confrontation suite.
Used downstream of zppy for ELM diagnostics.

- https://www.ilamb.org/
- Repo: https://github.com/rubisco-sfa/ILAMB

---

## Inspecting files at the command line

```bash
# Header only
ncdump -h <casename>.eam.h0.0001-01.nc | less

# A specific variable's values
ncdump -v PRECT <casename>.eam.h0.0001-01.nc | head -100

# CDO time mean
cdo timmean input.nc time_mean.nc

# NCO arithmetic
ncwa -a time -d lev,500.,500. input.nc out_500hPa.nc
ncks -v PRECT,TS input.nc small.nc

# Variable-by-variable diff
nccmp -df run_a.nc run_b.nc
```

For MPAS files (which use the unstructured cell index `nCells` rather
than `lat`/`lon`), use `paraview-mpas` or `mpas_analysis` for spatial
plots, or remap to a regular lat-lon grid first with
`MPAS-Tools/mpas_remap`.

---

## Short-term archiving

CIME has built-in short-term archiving that moves history files out of
`RUNDIR` (which lives on hot scratch) into `DOUT_S_ROOT` (warmer,
longer-lived storage):

```bash
./xmlchange DOUT_S=TRUE                          # enable
./xmlchange DOUT_S_ROOT=/path/to/archive
./case.st_archive                                # run manually after a job
```

Archived structure:

```
$DOUT_S_ROOT/$casename/
├── atm/hist/   ← <casename>.eam.h0.*.nc
├── atm/rest/
├── lnd/hist/
├── lnd/rest/
├── ocn/hist/
├── ocn/rest/
├── ice/hist/
├── ice/rest/
├── rof/hist/
├── rof/rest/
└── cpl/{hist,rest}/
```

zppy and `mpas_analysis` expect this archived layout. Set
`DOUT_S=TRUE` from the start of any production run.

---

## Where to next

- Performance considerations for I/O and PE layout: `performance-and-scaling.md`
- Diagnose corrupt or missing output: `debugging.md`
- Configure EAM-specific output: `eam-atmosphere.md`
- Configure MPAS streams: `mpas-ocean-seaice.md`
