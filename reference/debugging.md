# Debugging E3SM

A field guide to the most common E3SM failure modes: build errors,
runtime crashes, conservation violations, restart mismatches, and
non-bit-for-bit results. CIME hides a lot of complexity, so the first
step is almost always finding the right log file.

---

## The log files

Three classes of log file matter, all in `RUNDIR`:

| File | Source | What it has |
|------|--------|-------------|
| `cesm.log.<jobid>.<date>` | CIME job wrapper | Module loads, MPI launch, batch system messages, top-level traceback |
| `e3sm.log.<jobid>.<date>` | All MPI tasks merged | Coupled model stdout, MPI errors |
| `cpl.log.<jobid>.<date>` | Coupler driver | Per-step coupling diagnostics, conservation budgets |
| `atm.log`, `lnd.log`, `ocn.log`, `ice.log`, `rof.log`, `glc.log`, `wav.log` | Per-component stdout | Component-specific tracebacks and warnings |

After a successful run these are gzipped (`.log.<jobid>.<date>.gz`) and
moved into `$DOUT_S_ROOT/<casename>/logs/` if archiving is on.

To diagnose a failure, always start with `cesm.log` (was it a launch
problem?), then `e3sm.log` (was it inside the model?), then the
component logs.

---

## 1. Build failures

### `MODEL BUILD HAS FAILED` with no obvious error

```bash
./case.build 2>&1 | tee build_attempt.log
grep -i "error" build_attempt.log | head -50
grep -i "undefined reference" build_attempt.log
```

The first error is usually drowned in noise. Search **upward** from any
"failed" line for the first true compiler error.

### `cannot find -lmpi` / `cannot find -lmct`

A submodule is missing or stale. Verify:

```bash
cd /path/to/E3SM
git submodule status                # all should show a SHA, not blank
git submodule update --init --recursive --depth=1
```

Then reset and rebuild:

```bash
cd /path/to/case
./case.build --clean
./case.build
```

### CMake error: `Could NOT find ...`

A required system library (parallel HDF5, parallel netCDF) is not
loaded. On a supported machine, the per-machine module set in
`cime_config/machines/config_machines.xml` should load these
automatically. If not:

- Confirm you are on the actual machine, not a head node alias.
- Check `~/.cime/config_machines.xml` for personal overrides that may be
  hiding the machine-default modules.
- Run `module list` and compare to what `config_machines.xml` declares.

### EAMxx C++ compile error mentioning `Kokkos`

Almost always a stale `externals/ekat`. Refresh:

```bash
cd /path/to/E3SM/externals/ekat
git submodule update --init --recursive
cd /path/to/case
./case.build --clean
./case.build
```

### `Error: Symbol 'X' has already been used` (Fortran)

You have a SourceMods file shadowing a built-in module, but with a
duplicated subroutine name. Either rename your subroutine or remove the
SourceMods file.

### Build hangs at "Linking e3sm.exe" for >30 minutes

Login-node memory exhaustion or Lustre stall. On most DOE machines you
should build from a compute node:

```bash
salloc -N 1 -t 2:00:00 --account=<ALLOCATION>
./case.build
```

---

## 2. Runtime crashes

### `MPT ERROR: rank N exit signal 11` (segfault)

Usually a NaN propagated into a divide, or an out-of-bounds array access.
Rebuild in debug mode:

```bash
./xmlchange DEBUG=TRUE
./case.build --clean
./case.build
./case.submit
```

Then inspect `e3sm.log.*` for the traceback. With `DEBUG=TRUE` you get
file names and line numbers.

For floating-point exceptions, also enable FPE trapping (atmosphere only):

```bash
./xmlchange --append CAM_CONFIG_OPTS=-fpe-trap
```

For EAMxx, the `fpe` test build type traps FPEs:

```bash
cd components/eamxx
./scripts/test-all-eamxx -m <MACHINE> -t fpe
```

### `Error: missing variable in restart file`

You changed physics options between the original run and the restart,
and the new option requires a state variable the old restart doesn't
have. Either revert the option, or cold-start (`CONTINUE_RUN=FALSE`,
`./case.submit`).

### `mpiexec: Error: Failed to launch ... task`

Resource request mismatch. Check:

```bash
./xmlquery NTASKS,NTHRDS,MAX_MPITASKS_PER_NODE,COSTPES_PER_NODE
./xmlquery JOB_QUEUE,JOB_WALLCLOCK_TIME
```

Make sure `NTASKS_TOTAL <= max nodes for queue × cores per node`.

### `slurmstepd: error: Detected 1 oom-kill event`

Out of memory. Reduce per-rank memory pressure by:

- Increasing the number of MPI tasks (smaller per-rank domains)
- Using fewer threads per rank
- Reducing `mfilt` to write smaller history files
- For tracer-heavy chemistry runs, switching `PIO_TYPENAME` to a less
  memory-hungry backend

### Run hangs with no output

If `e3sm.log.*` shows the model started but then nothing, two possibilities:

1. Deadlock in coupling. Check `cpl.log.*` for the last successful coupling
   step. Compare against the coupler period (`./xmlquery NCPL_BASE_PERIOD,
   ATM_NCPL`).
2. I/O stall. Look at the timing report: if `CPL:RUN_LOOP` is hours
   while `CPL:ATM_RUN + CPL:OCN_RUN + ... ≪ that`, you are stuck in I/O.
   Reduce `mfilt` or change `PIO_NUMTASKS`.

---

## 3. Conservation problems

### Coupler budget shows non-zero residual

The coupler logs per-component water, energy, and salt budgets in
`cpl.log.*`. A non-zero "net" residual that grows over time indicates a
conservation bug in one of the component's flux exchange.

```bash
grep -A 20 "BUDGET DIAGNOSTICS" cpl.log.* | less
```

Common causes:

- New physics added without updating the coupler exchange (e.g., a new
  water reservoir but no compensating flux).
- Mismatch between `WCYCL` and a single-forcing experiment without
  updated SST file.
- Numerical error from very-low precision builds (`sp` for SCREAM).

### Sea ice mass not conserved

Frazil ice mass exchange between MPAS-Ocean and MPAS-Seaice has a
specific protocol (see `mpas-ocean-seaice.md` for the v2 update). If
you modified the frazil scheme, verify against the v2 paper (Golaz et al.
2022) which reports conservation to computational accuracy over 500
years.

---

## 4. Restart and bit-for-bit reproducibility

### `ERS` (exact restart) test

The standard CIME test for restart correctness:

```bash
cd cime/scripts
./create_test ERS.<resolution>.<compset>.<machine>
```

`ERS` runs the model twice: once for the full duration, once for half
the duration plus a restart for the second half. The resulting end-state
NetCDF files must be bit-identical.

If `ERS` fails, the most common causes:

- A stateful variable not written to or read from the restart file.
- An uninitialized variable that takes different values on first
  iteration of a fresh run vs. a restart run.
- A loop that depends on accumulated state outside of the model's
  prognostic vector.

### `PEM` (PE count) test

Runs the same case with two different MPI task counts; results must be
bit-identical:

```bash
./create_test PEM.<resolution>.<compset>.<machine>
```

If `PEM` fails, you have a parallelism bug: a global reduction that
depends on summation order, an MPI-unfriendly random number generator,
or a halo exchange that's not properly synchronized.

### `PET` (PE thread) test

Same idea but varying the OpenMP thread count:

```bash
./create_test PET.<resolution>.<compset>.<machine>
```

If `PET` fails, you have a threading bug. Common in EAMxx Kokkos code if
a parallel-for has a race condition.

### `SMS_D` (debug smoke test)

Smoke test built with `DEBUG=TRUE`:

```bash
./create_test SMS_D.<resolution>.<compset>.<machine>
```

If a normal `SMS` passes but `SMS_D` fails, you likely have an
uninitialized variable, an out-of-bounds access, or an FPE that gets
masked at -O2 but trips at -O0 with checks on.

---

## 5. EAMxx-specific debugging

EAMxx has its own standalone test harness (no CIME, no driver):

```bash
cd components/eamxx
./scripts/test-all-eamxx -m <MACHINE>             # all build types
./scripts/test-all-eamxx -m <MACHINE> -t dbg      # debug only
./scripts/test-all-eamxx -m <MACHINE> -t fpe      # FPE trapping on
./scripts/test-all-eamxx --baseline-dir=LOCAL     # compare to local baselines
```

These tests are much faster than CIME-based tests. Use them as the inner
debug loop when you are working on EAMxx C++ source.

If you are debugging a specific Kokkos kernel, build with
`Kokkos_ENABLE_DEBUG_BOUNDS_CHECK=ON` to catch out-of-bounds device-side
accesses.

---

## 6. Helpful diagnostics

### Dump a NetCDF history file at the command line

```bash
ncdump -h <case>.eam.h0.0001-01.nc | less                    # header
ncdump -v PRECT <case>.eam.h0.0001-01.nc | head -100          # variable
ncwa -a time -d lev,500.,500. in.nc out_500hPa.nc             # vertical slice
```

### Compare two output files variable-by-variable

```bash
nccmp -df run_a.nc run_b.nc                        # data + header diff
cdo diff run_a.nc run_b.nc                         # CDO equivalent
```

### Inspect the case status

```bash
cd $CASEROOT
./xmlquery --partial RUN                           # all RUN_* settings
./xmlquery EXEROOT,RUNDIR,DOUT_S_ROOT
cat README.case                                    # human-readable summary
ls CaseDocs/                                       # documented namelists
```

### Inspect the build status

```bash
cat $EXEROOT/CMakeCache.txt | grep -i "BUILD_TYPE\|MPI\|COMPILER"
ls $EXEROOT/cmake-bld
```

### Find which file you really need to edit

```bash
cd /path/to/E3SM
git grep -l 'subroutine_or_variable_name' components/eam/src/ | head
git grep -l 'subroutine_or_variable_name' components/eamxx/src/ | head
```

### Reading test results

```bash
cd /path/to/test/area
ls TestStatus.log                                  # human-readable status
cat TestStatus                                     # machine-readable
tail TestStatus.log                                # last few lines (PASS/FAIL)
```

---

## Where to next

- Pick the right test compset to reproduce a bug: `running-cases.md`
- Add an integration test for your fix: `contributing-pr.md`
- Tune PE layout to avoid OOM: `performance-and-scaling.md`
- Inspect output to confirm the fix: `output-and-postprocess.md`
