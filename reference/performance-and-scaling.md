# Performance and Scaling

E3SM is a parallel application that runs on tens to tens of thousands of
nodes on DOE Leadership Computing Facilities. Getting good performance
requires picking the right PE (processor element) layout, the right
threading, and on heterogeneous systems, the right GPU configuration.
This page covers the knobs and the per-machine considerations.

---

## PE layouts

A **PE layout** assigns each component to a contiguous set of MPI tasks
(and optionally OpenMP threads). The seven slots (ATM, LND, ICE, OCN,
ROF, GLC, WAV) can run sequentially (sharing the same processors), in
parallel (each on its own processor block), or as a hybrid.

CIME ships pre-defined PE layouts addressed by `--pecount`:

| `--pecount` | Approximate scale | Use case |
|-------------|-------------------|----------|
| `XS` | ~1 node | Smoke tests on `ne4` |
| `S` | a few nodes | Quick development cycles |
| `M` | tens of nodes | Standard low-res development |
| `L` | hundreds of nodes | Production low-res |
| `X1`, `X2` | extra-large | High-res or RRM |
| `custom-N` | N nodes (E3SM extension) | Use `run_e3sm.template.sh` to set explicitly |

The actual mapping for each `(compset, resolution, machine)` combination
lives in `cime_config/allactive/config_pesall.xml` (coupled) and
component-specific `config_pes.xml` files.

After `create_newcase`, inspect the assigned layout:

```bash
./xmlquery NTASKS,NTHRDS,ROOTPE
./xmlquery --partial NTASKS                     # all NTASKS_* settings
./pelayout                                      # human-readable summary
```

Modify:

```bash
./xmlchange NTASKS_ATM=2048,NTASKS_OCN=1024,NTASKS_ICE=512
./xmlchange NTHRDS_ATM=2,NTHRDS_OCN=2
./xmlchange ROOTPE_OCN=2048                     # OCN starts at PE 2048

# After any PE layout change, MUST do:
./case.setup --reset
./case.build
```

---

## Custom layout via `run_e3sm.template.sh`

`run_e3sm.template.sh` includes a `custom_pelayout()` function that
maps a node count to NTASKS for several E3SM machines:

```bash
# In the template script, set:
readonly PELAYOUT="custom-128"          # 128 nodes

# Internally the script looks up cores per node:
#   pm-cpu     → 128
#   chrysalis  → 64
#   compy      → 40
#   anvil      → 36
# and computes NTASKS = nnodes * cores_per_node, NTHRDS=1
```

To use this on a different machine, add a corresponding `if` branch.

---

## Threading (OpenMP)

Each component can be threaded with OpenMP via `NTHRDS_<comp>`. Total
hardware tasks per node is then `MAX_MPITASKS_PER_NODE / NTHRDS`, with
the remainder being OpenMP threads.

Example on Perlmutter CPU (128 cores per node, 2 sockets × 64):

```bash
./xmlchange MAX_MPITASKS_PER_NODE=64        # 64 MPI ranks per node
./xmlchange NTHRDS_ATM=2,NTHRDS_LND=2       # 2 OpenMP threads per rank
./xmlchange NTHRDS_OCN=1                    # ocean is MPI-only
```

Thread placement matters: bind threads to physical cores with the
machine's preferred mechanism (Slurm `--cpu-bind`, OpenMP
`OMP_PLACES=cores`, etc.). The per-machine launch command in
`config_machines.xml` typically handles this.

EAMxx is heavily Kokkos-based; on CPU it uses Kokkos OpenMP backend, on
GPU it uses Kokkos CUDA / HIP / SYCL.

---

## GPU support

E3SM supports GPUs through two layers:

1. **EAMxx (SCREAM)**: Kokkos-based, performance-portable across NVIDIA
   (CUDA), AMD (HIP/ROCm), and Intel (SYCL) GPUs. This is the primary
   GPU code path in E3SM v3.
2. **EAM (Fortran)**: Selected hot kernels via YAKL
   (`externals/YAKL`) and OpenACC. Coverage is partial; the Fortran
   atmosphere does not run end-to-end on GPU.

GPU machines and how they map to Kokkos backends:

| Machine | GPU | Backend | Per-machine cmake macros |
|---------|-----|---------|--------------------------|
| Perlmutter (`pm-gpu`) | NVIDIA A100 | CUDA | `cime_config/machines/cmake_macros/*pm-gpu*.cmake` |
| Frontier (`frontier`) | AMD MI250X | HIP | `cime_config/machines/Depends.crayclanggpu.cmake`, `craygnu-hipcc.cmake` |
| Aurora (`aurora`) | Intel Ponte Vecchio | SYCL via Kokkos | (machine-specific) |
| Polaris (`polaris`) | NVIDIA A100 | CUDA | (pre-Aurora dev) |
| Sunspot (`sunspot`) | Intel Ponte Vecchio | SYCL | (Aurora staging) |
| Summit-class (legacy) | NVIDIA V100 | CUDA | retired |

For an EAMxx GPU build:

```bash
./create_newcase \
    --case ../../cases/screamgpu \
    --compset F2010-SCREAMv1 \
    --res ne30pg2_ne30pg2 \
    --mach pm-gpu \
    --pecount M \
    --project <ALLOCATION>

# CIME automatically picks the GPU-flavored CMake macros for pm-gpu.
```

To verify your build will use GPU:

```bash
./xmlquery GPU_TYPE,GPU_OFFLOAD,KOKKOS_OPTIONS
```

---

## Per-machine notes

### Perlmutter (NERSC)

- CPU partition: `pm-cpu`, AMD EPYC Milan, 128 cores/node.
- GPU partition: `pm-gpu`, 4× NVIDIA A100 / node, 64 cores + 4 GPUs.
- Filesystems: `$PSCRATCH` (Lustre, hot, 60-day purge), `$CFS`
  (community file system, persistent).
- Set `CIME_OUTPUT_ROOT=$PSCRATCH/E3SMv3` for build/run.
- Modules: `PrgEnv-gnu` (default for E3SM), `cray-mpich`, `cray-hdf5-parallel`,
  `cray-netcdf-hdf5parallel`, `cray-parallel-netcdf`.

### Frontier (OLCF)

- AMD EPYC Trento + 4× MI250X / node = 8 GPU compute dies / node.
- Use HIP backend in Kokkos.
- Filesystem: `$ORLY_LUSTRE/scratch/<user>/`.
- Compilers: Cray Clang (`crayclang`), Cray GNU (`craygnu`), Cray HIPCC.
- For EAMxx GPU runs use the `craygnu-hipcc` toolchain.

### Aurora (ALCF)

- Intel Sapphire Rapids + 6× Ponte Vecchio (Intel Data Center GPU Max) /
  node.
- SYCL via Kokkos for EAMxx.
- New machine, support actively maturing.

### Chrysalis (LCRC, ANL)

- Intel Cascade Lake, 64 cores/node, no GPU.
- Workhorse for E3SM development. Faster turnaround than Perlmutter for
  small-to-mid jobs.
- Filesystem: `/lcrc/group/e3sm/`.

### Compy (PNNL)

- Intel Xeon, 40 cores/node, no GPU.
- Used by PNNL E3SM developers. Documented in `config_machines.xml`.

### Anvil and Improv (LCRC)

- Anvil: 36 cores/node, no GPU. Smaller jobs.
- Improv: newer LCRC system.

---

## I/O performance: SCORPIO

E3SM's parallel I/O goes through **SCORPIO**
(`externals/scorpio`), a modern PIO replacement. SCORPIO can use:

- **PnetCDF** for parallel writes to a single shared file
- **NetCDF4 / HDF5** parallel for chunked writes
- **ADIOS2** for very high-throughput log-structured output (supported
  on some machines)

To inspect or change the backend:

```bash
./xmlquery PIO_TYPENAME                      # e.g., pnetcdf, netcdf4p, adios
./xmlchange PIO_TYPENAME=pnetcdf
./xmlquery PIO_NUMTASKS                      # number of I/O tasks
./xmlchange PIO_NUMTASKS=64
./xmlquery PIO_STRIDE                        # stride between I/O tasks
```

For high-resolution runs (`ne120` or `ne256`), tune `PIO_NUMTASKS` and
`PIO_STRIDE` to match your machine's optimal I/O concurrency. Too few
I/O tasks bottleneck on a single Lustre OST; too many overwhelm the
metadata server.

---

## Timing diagnostics

Every CIME run produces a timing report:

```
$RUNDIR/timing/e3sm_timing.<casename>.<jobid>
$RUNDIR/timing/e3sm_timing_stats.<casename>.<jobid>
```

The first file shows wall-clock cost per component and per coupling
phase; the second has detailed call-tree statistics. Look for:

- `CPL:RUN_LOOP`, total run time
- `CPL:ATM_RUN`, `CPL:OCN_RUN`, ..., per-component
- `a:[component]_run_loop`, inner per-component loop
- `CPL:WAIT_*`, coupler waits (idle time → load imbalance)

If `WAIT_OCN` is large, the ocean is the bottleneck → give it more
tasks. If `WAIT_ATM` is large, the atmosphere is. The goal is roughly
equal `RUN` times across active components.

---

## Compile-time vs run-time perf

| Knob | Where | Effect |
|------|-------|--------|
| `DEBUG=TRUE` | `xmlchange` before `case.build` | -O0, bounds checks, ~5-10x slower |
| `DEBUG=FALSE` | default | Production optimization (-O2/-O3) |
| `CAM_CONFIG_OPTS` | `xmlchange` | EAM build options (e.g., `-cosp`, MMF flags) |
| `CMAKE_BUILD_TYPE` | machine cmake macros | release vs debug |
| `MPILIB` | machine | MPI implementation (cray-mpich, openmpi, intelmpi) |
| `COMPILER` | machine | gnu, intel, cray, nvidia |

For benchmarking, **always** use a release build (`DEBUG=FALSE`) with
SCORPIO output enabled but with `mfilt` tuned to write a small number of
files per segment so the I/O cost is realistic.

---

## CI test machines

Two machines are heavily used by automated CI:

| Machine | Hardware | Purpose |
|---------|----------|---------|
| `mappy` | small SNL workstation | Quick CI |
| `ghci-oci` | Oracle Cloud, GitHub-hosted | Public CI for `eamxx-sa-testing.yml` and `eamxx-v1-testing.yml` |
| `weaver` | small SNL workstation | EAMxx CI |

Build types tested:
- `sp`, single precision
- `dbg`, debug
- `fpe`, floating-point exceptions on
- `opt`, optimized

Nightly results: https://my.cdash.org/index.php?project=E3SM

---

## Where to next

- Submit a job through CIME: `running-cases.md`
- Tune output for I/O performance: `output-and-postprocess.md`
- Diagnose a hang or crash: `debugging.md`
- Submit a performance PR: `contributing-pr.md`
