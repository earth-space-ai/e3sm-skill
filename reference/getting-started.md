# Getting Started with E3SM

This guide walks from zero to a successful `case.build` on a supported DOE
machine. After this, see `running-cases.md` to actually launch a simulation
and `architecture.md` to learn what the components are.

E3SM is not a self-contained desktop application. It is a coupled climate
model designed for DOE Leadership Computing Facilities. It can be built on
a personal Linux box for a tiny low-resolution test, but every scientifically
meaningful configuration requires HPC-class hardware.

---

## Step 1: Decide where you will run

Before you clone, confirm you have an account and an allocation on one of
the supported machines.

| Class | Machine | Center | Notes |
|-------|---------|--------|-------|
| Production | `pm-cpu`, `pm-gpu` | NERSC Perlmutter | CPU and GPU partitions |
| Production | `frontier` | OLCF | Exascale, AMD MI250X via HIP |
| Production | `aurora` | ALCF | Intel GPUs via SYCL/Kokkos |
| Production | `chrysalis` | LCRC, ANL | Workhorse for E3SM development |
| Production | `compy` | PNNL | Pacific Northwest National Lab |
| Mid | `anvil`, `improv` | LCRC, ANL | Smaller jobs |
| Dev | `polaris`, `sunspot`, `crux` | ALCF | Pre-Aurora dev systems |
| Test | `linux-generic`, `WSL2`, `mac`, `singularity` | Local | Tiny ne4 cases only |

The full list lives in `cime_config/machines/config_machines.xml` (entries
like `<machine MACH="pm-cpu">`). Run

```bash
cd cime/scripts
./query_config --machines
```

after cloning to enumerate every machine the local checkout knows about.

If your target machine is **not** in that list, you must port E3SM by
following the
[CIME Porting Guide](https://esmci.github.io/cime/versions/master/html/users_guide/porting-cime.html).
Porting is a multi-day effort (system modules, compilers, batch system,
input data root, baseline directory) and is out of scope here.

---

## Step 2: Clone with all submodules

E3SM tracks roughly twenty git submodules: CIME, MCT, scorpio,
scorpio_classic, EKAT, YAKL, cub, ninja, mam4xx, FATES, MPP, sBeTR,
RRTMGP (rte-rrtmgp), COSPv2, MARBL, Icepack, CVMix, GOTM, FFTW, SHTNS,
PPR, WW3, GIAC, SeaLevelModel. Almost all early build failures trace back
to a missing or stale submodule.

```bash
# Recommended: clone with submodules in one shot
git clone --recursive https://github.com/E3SM-Project/E3SM.git
cd E3SM

# Or, after a non-recursive clone:
git submodule update --init --recursive --depth=1
```

Verify:

```bash
ls cime/scripts                              # → create_newcase, query_config, ...
ls externals/scorpio                         # should not be empty
ls externals/mam4xx                          # should not be empty
ls components/elm/src/external_models/fates  # should not be empty
ls components/mpas-ocean/src/MARBL           # should not be empty
ls components/mpas-seaice/src/icepack        # should not be empty
```

If any of these are empty, re-run `git submodule update --init --recursive`.

The `--depth=1` flag keeps each submodule shallow, which saves several GB
on a fresh clone. If you need full history of, say, `cime/`, fetch it
later: `cd cime && git fetch --unshallow`.

The E3SM project is migrating to a wrapper called **git-fleximod**
(`git fleximod update`) that drives the same submodule update with a
single command and respects per-component branch pinning specified in
`.gitmodules`. Both invocations work; pick whichever your team uses.

---

## Step 3: Set up your environment on the target machine

On a supported machine, the per-machine module environment is encoded in
`config_machines.xml` and loaded automatically by `case.setup` and
`case.build`. You do not need to `module load` anything by hand for the
case workflow. You **do** need:

| Setting | Where it lives |
|---------|----------------|
| `DIN_LOC_ROOT` (input data) | `config_machines.xml` per machine |
| `CIME_OUTPUT_ROOT` (case output) | `config_machines.xml` per machine; can override via `xmlchange` |
| `BASELINE_ROOT` (test baselines) | `config_machines.xml` per machine |
| Project / allocation | Set with `--project` on `create_newcase`, or default account from your batch system |

Personal overrides go in `~/.cime/config_machines.xml` (and
`~/.cime/config.xml` for non-machine settings). This is the right place
to set, e.g., `CIME_OUTPUT_ROOT` to a scratch directory that is not the
machine default.

---

## Step 4: Build prerequisites

On a supported machine, all of the following are pre-installed and loaded
via the per-machine module set:

- **CMake ≥ 3.18** (top-level `components/CMakeLists.txt`)
- **MPI** (Cray-MPICH on Perlmutter/Frontier, MPICH on Aurora, OpenMPI on Chrysalis)
- **Fortran compiler** (Intel `ifx`/`ifort`, GNU `gfortran`, Cray `ftn`, NVIDIA `nvfortran`)
- **C++ compiler** with C++17 (Intel, GNU, Cray, NVIDIA, Clang)
- **HDF5 + parallel netCDF + PnetCDF** (used by SCORPIO)
- **PIO/SCORPIO** (built in-tree from `externals/scorpio`)
- **Python ≥ 3.6** (CIME, test infrastructure)
- **Perl** (CIME also relies on Perl scripts)
- For GPU builds: **CUDA** (Perlmutter), **HIP/ROCm** (Frontier), or
  **SYCL/Kokkos-Intel** (Aurora)

For a personal Linux machine with `linux-generic`, see
https://docs.e3sm.org/E3SM/installation/ and the CIME Porting Guide.

---

## Step 5: First case (smoke test)

The smallest meaningful coupled E3SM run is a 5-day `WCYCL1850NS` case at
the `ne4pg2_r05_oQU480` resolution. This compsets uses no spun-up initial
conditions and is intended only as a regression test, but it runs in
minutes on most machines.

```bash
cd cime/scripts
./create_newcase \
    --case ../../cases/smoketest \
    --compset WCYCL1850NS \
    --res ne4pg2_r05_oQU480 \
    --mach <YOUR_MACHINE> \
    --pecount S \
    --project <YOUR_ALLOCATION>

cd ../../cases/smoketest

# Configure
./case.setup

# Build the executable (out of source; goes to $EXEROOT)
./case.build

# Verify
./xmlquery EXEROOT             # path to build dir
./xmlquery RUNDIR              # path to run dir
ls $(./xmlquery --value EXEROOT)/e3sm.exe

# Run a 5-day test
./xmlchange STOP_OPTION=ndays,STOP_N=5
./case.submit
```

`case.submit` queues the job through the machine's batch system (Slurm on
Perlmutter/Chrysalis/Compy, PBS on Polaris, LSF on Summit-class systems).
After it completes, history files appear in `RUNDIR` and short-term
archived data (if `DOUT_S=TRUE`) in `DOUT_S_ROOT`.

If `case.build` fails, see `debugging.md`.

---

## Step 6: What success looks like

At the end of `case.build`:

```
MODEL BUILD HAS FINISHED SUCCESSFULLY
```

After `case.submit` completes:

```
Run SUCCESSFUL since coupler time-step ...
```

In `RUNDIR` you should find files like:

```
<casename>.eam.h0.0001-01.nc          # EAM monthly history (or .eamxx. for SCREAM)
<casename>.elm.h0.0001-01.nc          # ELM monthly history
<casename>.mosart.h0.0001-01.nc       # MOSART monthly history
<casename>.mpaso.hist.am.timeSeriesStatsMonthly.0001-01-01.nc
<casename>.mpassi.hist.am.timeSeriesStatsMonthly.0001-01-01.nc
<casename>.cpl.hi.0001-01-01-00000.nc # coupler history (every HIST_N units)
e3sm.log.<jobid>.<date>               # main coupled stdout
cesm.log.<jobid>.<date>               # CIME-level stdout
cpl.log.<jobid>.<date>                # coupler timing/diagnostics
atm.log, lnd.log, ocn.log, ice.log, rof.log, glc.log, wav.log
```

The `.log` files are gzipped after a successful run
(`*.log.<jobid>.<date>.gz`).

---

## Common install pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `externals/scorpio` is empty | Forgot `--recursive` | `git submodule update --init --recursive --depth=1` |
| `case.build` fails with "no rule to make target" in MARBL | `components/mpas-ocean/src/MARBL` empty | Same as above |
| `Compset/grid combination not supported` | Compset and grid mismatch (e.g., `F1850` with `T62_oQU240`) | Use `query_config --grids` and check per-component support |
| `Machine X not found` | Not on a supported host or typo in `--mach` | `query_config --machines` |
| `Permission denied` writing to `$CIME_OUTPUT_ROOT` | Default dir is on shared scratch you do not own | Set `CIME_OUTPUT_ROOT` in `~/.cime/config_machines.xml` to your own scratch |
| Build hangs at `Linking e3sm.exe` for >30 min | Login-node OOM or filesystem stall on Lustre | Build from a compute node (`salloc` then `case.build`) |
| `module: command not found` | You launched outside the machine's normal shell init | Source `/etc/profile` or `module load PrgEnv-*` manually |
| EAMxx C++ compile error mentioning `Kokkos` | `externals/ekat` empty or stale | Re-run submodule update; `cd externals/ekat && git submodule update --init --recursive` |

For ongoing FAQs, see https://github.com/E3SM-Project/E3SM/discussions and
https://acme-climate.atlassian.net/wiki/spaces/DOC/overview.

---

## Where to next

- Map the components and their relationships: `architecture.md`
- Create your first scientifically meaningful case: `running-cases.md`
- Inspect output and run the standard diagnostics: `output-and-postprocess.md`
- Diagnose a build or run failure: `debugging.md`
