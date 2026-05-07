# MPAS-Ocean and MPAS-Seaice

E3SM's ocean and sea ice components share the MPAS (Model for Prediction
Across Scales) framework: an unstructured Spherical Centroidal Voronoi
Tessellation (SCVT) mesh that allows local refinement without nested
grids. Both components live in `components/mpas-ocean/` and
`components/mpas-seaice/`, with shared infrastructure in
`components/mpas-framework/`.

---

## The MPAS Voronoi mesh

Each ocean cell is a Voronoi polygon, typically a hexagon, with cell
centers and edges connecting neighbors. The mesh has two duals:

- **Primal mesh**: Voronoi cells (hexagons or other n-gons). Sea ice and
  ocean concentrations, volumes, tracers live here.
- **Dual mesh**: Delaunay triangles connecting the cell centers. Velocity
  components live at primal-cell vertices (= dual-cell centers).

This dual structure provides a convenient local coordinate system at every
edge: the line connecting two primal cell centers is orthogonal to the
edge connecting the two adjacent dual-mesh cell centers.

Velocity components are aligned with a locally Cartesian east/north
coordinate system rotated so the pole of the rotation lies on the equator
at 0° longitude (avoids polar singularity).

The default E3SMv3 ocean+ice mesh is **`IcoswISC30E3r5`**:
- Derived from the dual of a 30 km icosahedral mesh
- "wISC" suffix = with ice shelf cavities under the Antarctic ice shelves
- "E3r5" = E3SMv3 revision r5

---

## MPAS-Ocean

`components/mpas-ocean/`

### Governing equations

MPAS-Ocean solves the Boussinesq, hydrostatic, primitive equations on the
SCVT mesh. The key references in `components/mpas-ocean/docs/`:

- Ringler, T., Petersen, M., Higdon, R.L., Jacobsen, D., Jones, P.W.,
  Maltrud, M. (2013). A multi-resolution approach to global ocean
  modeling. *Ocean Modelling* 69, 211-232.
- Petersen, M.R., Jacobsen, D.W., Ringler, T.D., Hecht, M.W., Maltrud,
  M.E. (2015). Evaluation of the arbitrary Lagrangian-Eulerian vertical
  coordinate method in MPAS-Ocean. *Ocean Modelling* 86, 93-113.
- Petersen, M.R. et al. (2019). An evaluation of the ocean and sea ice
  climate of E3SM using MPAS and interannual CORE-II forcing. *JAMES* 11.

### Vertical coordinate

E3SM v2/v3 uses **z*** (z-star), an arbitrary Lagrangian-Eulerian (ALE)
displaced sea-surface coordinate. The ocean is divided into ~80 vertical
layers; the top layer thickness varies with the free surface.

### Time stepping

Split-explicit barotropic and baroclinic:

- **Baroclinic** time step: large, set by internal wave / CFL constraints.
- **Barotropic** time step: small, set by external (gravity wave) CFL.
  The barotropic mode is sub-cycled inside each baroclinic step.

This split saves substantial compute compared to a fully implicit or
fully explicit scheme.

### Vertical mixing: CVMix

Vertical mixing of momentum and tracers is provided by the **CVMix**
library (`components/mpas-ocean/src/cvmix`), which packages KPP, Richardson
number based mixing, and others. GOTM (`components/mpas-ocean/src/gotm`)
is also available for second-moment turbulence closures.

### Biogeochemistry: MARBL

E3SM's modern ocean biogeochemistry is **MARBL** (Marine Biogeochemistry
Library), included as a submodule under `components/mpas-ocean/src/MARBL`.
MARBL provides:

- Multi-element prognostic ecosystem (C, N, P, Fe, Si)
- Multiple phytoplankton functional types
- Carbonate chemistry and air-sea CO2 flux
- Dissolved oxygen, alkalinity, nitrification, denitrification

The older E3SM ocean BGC (`components/mpas-ocean/src/BGC`) remains in the
tree for legacy compatibility.

### Other in-tree externals

| Path | Purpose |
|------|---------|
| `src/cvmix` | Community Vertical Mixing |
| `src/gotm` | General Ocean Turbulence Model |
| `src/SHTNS` | Spherical harmonic transforms |
| `src/FFTW` | Fast Fourier transforms |
| `src/ppr` | Piecewise Parabolic Reconstruction |

### Standalone vs coupled

MPAS-Ocean can run as a standalone executable using its own driver, or as
a component inside the E3SM coupled system. In the coupled case the CIME
`case.build` constructs `e3sm.exe` which calls the MPAS-Ocean library
through the driver-mct (or driver-moab) coupler.

### Test cases: Polaris and Compass

Standalone test cases (idealized configurations, regression tests, and
mesh generation) live in two external repos:

- **Polaris** (https://github.com/E3SM-Project/polaris), modern Python
  framework, current development home
- **Compass** (https://github.com/MPAS-Dev/compass), older framework

Documentation: http://docs.e3sm.org/polaris/main/ and
https://mpas-dev.github.io/compass/latest/.

### Omega: the next-generation ocean

Omega is a planned C++/Kokkos rewrite of MPAS-Ocean for performance
portability. Documentation lives at
https://docs.e3sm.org/Omega/develop/index.html. Omega is **not** yet a
supported E3SM ocean component; MPAS-Ocean remains the production
choice.

---

## MPAS-Seaice

`components/mpas-seaice/`

The sea ice component, on the same MPAS mesh as the ocean (so
ocean-to-ice mapping is trivial vertex-to-cell). Ice concentration,
volume, and tracers at cell centers; velocity at cell vertices using
B-grid discretization.

### Code structure

| Directory | Function |
|-----------|----------|
| `bld` | Namelist configuration files |
| `cime_config` | Build and configuration scripts |
| `docs` | Documentation |
| `driver` | Coupling modules |
| `src` | Source code (model physics, output) |
| `src/analysis_members` | Source code for diagnostic output streams |
| `src/icepack` | Submodule link to Icepack column physics |
| `src/model_forward` | Top-level mpas-seaice modules (driver) |
| `src/shared` | Mesh, constants, and other general-purpose modules |
| `testing` | Testing scripts |

Some shared MPAS-Seaice routines live in
`components/mpas-framework/core_seaice`.

### Velocity solver

The sea ice momentum equation (Hibler 1979; Hunke & Dukowicz 1997) is
solved using an EVP (elastic-viscous-plastic) approach. Two schemes
compute strain-rate tensor and divergence of internal stress on MPAS
meshes:

- **Variational** scheme (based on CICE; Hunke & Dukowicz 2002), using
  Wachspress basis functions (Dasgupta 2003) integrated with Dunavant
  (1985) quadrature.
- **Weak** scheme using line-integral forms.

### Horizontal transport: incremental remapping

Ice area, volume, and tracers are advected with an **incremental
remapping** scheme generalized from Dukowicz & Baumgardner (2000) and
Lipscomb & Hunke (2004) to work on hexagonal Voronoi cells. The IR
scheme handles tracers up to type 3 (e.g., aerosols on snow on ice).

### Column physics: Icepack

`components/mpas-seaice/src/icepack` is a submodule of the **Icepack**
library maintained by the CICE Consortium. E3SM uses the column physics
portion of Icepack; Icepack's own driver and testing scripts are used
when preparing E3SM modifications to merge back upstream.

Icepack provides:

| Package | Implementation |
|---------|----------------|
| Vertical thermodynamics | Bitz & Lipscomb (1999), Turner et al. (2013), Turner & Hunke (2015) |
| Default thermodynamics | Mushy-layer (Turner et al. 2013), sea ice as two-phase ice + brine system |
| Radiation | Delta-Eddington with SNICAR-AD (Briegleb & Light 2007; Dang et al. 2019) |
| Melt ponds (default) | Level-ice scheme (Hunke et al. 2013) |
| Melt ponds (alt) | "sealvl" hypsometric scheme |
| Melt ponds (alt 2) | "topo" hydrostatic scheme |
| Snow morphology | Multi-layer (5-layer in v2+) with grain radius evolution, wind redistribution |
| Mechanical redistribution | Lipscomb et al. (2007) ridging |
| Thickness-space transport | Lipscomb (2001) |

E3SM-specific Icepack docs:
https://e3sm-icepack.readthedocs.io/en/latest/. Source for that
documentation lives in the E3SM Icepack fork:
https://github.com/E3SM-Project/Icepack/.

### Mushy-layer thermodynamics in detail

The default vertical thermodynamics scheme is the **mushy layer**
formulation. Sea ice is treated as a two-phase system of crystalline fresh
ice and liquid brine. Enthalpy depends on temperature and salinity (both
prognostic). Equations derive from:

1. Conservation of energy
2. Conservation of salt
3. Ice-brine liquidus relation (T-S phase diagram)
4. Darcy flow through a porous medium for vertical brine transport

When the ice is cold, brine pockets are isolated. Warmer temperatures
expand and connect them into vertical channels through which meltwater,
seawater, and biology can move.

### Snow physics (E3SMv2 update)

A new snow morphology was added in E3SMv2 with:

- Wind-driven redistribution (losses to leads and melt ponds, piling
  against ridges)
- Snow grain radius as a prognostic tracer, evolving with temperature
  gradient and wet metamorphism
- Five vertical snow layers (replacing the single-layer of v1) atop each
  of the five ice thickness categories
- Direct feedback to SNICAR-AD radiation

Fresh snow falls at 54.5 μm grain radius; dry maximum is 2800 μm.

### Biogeochemistry

Icepack incorporates a comprehensive sea ice BGC:

- Two aerosol options
- Water isotopes
- Vertically resolved BGC tracers (algae, nutrients, inorganic carbon, ...)

Complexity is configured through Icepack namelist flags. Documentation in
the Icepack readthedocs.

### Coupling to the ocean

From the v1 description (Golaz et al. 2019), refined in v2 (Golaz et al.
2022):

- Sea ice mass contributes to the ocean barotropic mode (visible at the
  free surface over continental shelves).
- In shallow water, the weight is limited to prevent column evacuation.
- Frazil ice formed in MPAS-Ocean is passed to MPAS-Seaice as crystal
  volume at fixed 4 PSU salinity.
- The ocean temperature immediately under the ice equals the mushy
  liquidus basal temperature (Turner & Hunke 2015), not a fixed -1.8°C.
- v2 conserves frazil heat and mass to computational accuracy over a
  500-year control through additional adiabatic mass exchange between
  upper ocean and ice with assumed 25% mushy liquid content.

### Prescribed-ice mode

For AMIP-style F-compset runs that need ice surface fluxes but not a
fully prognostic cover, MPAS-Seaice supports a prescribed-ice mode
(modeled on the CESM implementation):

- Ice dynamics disabled; thermodynamics still active.
- At each step, ice area is interpolated in time and space from an input
  data set; ice thickness is reset to 2 m (NH) or 1 m (SH).
- Snow volume preserves prognosed snow thickness from the previous step.
- Snow temperatures reset to surface temperature; ice temperatures
  interpolated linearly between surface and freezing temperature at base.
- Vertical ice salinity profile reset to Bitz & Lipscomb (1999).

This mode does **not** conserve energy or mass and is for atmosphere
sensitivity experiments only.

---

## Compsets that exercise these components

| Compset | Ocean | Sea ice | Used for |
|---------|-------|---------|----------|
| `WCYCL1850` | MPASO active | MPASSI active | Production coupled, water cycle |
| `CRYO1850` | MPASO active with ISC | MPASSI active | Polar processes, sea-level rise |
| `CMPASO-NYF` | MPASO active | DICE | Ocean-only with normal-year forcing |
| `DTESTM` | DOCN | MPASSI active | Sea ice testing |
| `F1850`, `F2010`, `F20TR` | DOCN | DICE (or MPASSI prescribed) | Atmosphere-only |
| `I*` series | SOCN | SICE | Land-only |

---

## Output

MPAS-Ocean and MPAS-Seaice write their own NetCDF history streams
(separate from the EAM/ELM `.h0.nc` convention). Streams are configured
through XML files generated at case setup. The most common streams:

- `<casename>.mpaso.hist.am.timeSeriesStatsMonthly.YYYY-MM-01.nc`, monthly
  ocean diagnostics
- `<casename>.mpassi.hist.am.timeSeriesStatsMonthly.YYYY-MM-01.nc`, monthly
  sea ice diagnostics
- `<casename>.mpaso.rst.YYYY-MM-DD_00000.nc`, ocean restart
- `<casename>.mpassi.rst.YYYY-MM-DD_00000.nc`, sea ice restart

The standard postprocessing tool for MPAS components is **mpas_analysis**
(https://github.com/MPAS-Dev/MPAS-Analysis). See
`output-and-postprocess.md`.

---

## Where to next

- Atmosphere details (EAM/EAMxx): `eam-atmosphere.md`
- Inspect output and run mpas_analysis: `output-and-postprocess.md`
- Tune ocean PE layout for performance: `performance-and-scaling.md`
- Submit a MPAS-Ocean or MPAS-Seaice PR: `contributing-pr.md`
