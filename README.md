# E3SM Skill

A progressive-disclosure skill for the [E3SM (Energy Exascale Earth System
Model)](https://github.com/E3SM-Project/E3SM), the U.S. Department of
Energy's flagship coupled climate model.

> **Maintainer of E3SM:** E3SM Project, U.S. Department of Energy
> **Skill author:** Koutian Wu (ktwu01@gmail.com)
> **Skill version:** 0.1.0

> ⚠️ **Disclaimer — please read before using this skill.**
> This skill is **not a gold-standard reference**. It is a helper that lowers
> the barrier for new users to **get their hands dirty** with the model. AI
> agents (and the humans drafting this material) make mistakes; commands, file
> paths, namelist options, and physics explanations here can be wrong,
> incomplete, or out of date. **Always cross-check with the official model
> documentation, the source code, and a human expert before trusting any
> output for research, publication, or operational use.**

## What This Is

A self-contained knowledge package that teaches AI agents (and humans) how
to **clone, build, configure, run, analyze, debug, and contribute to** E3SM,
covering:

- the coupled atmosphere-land-ocean-sea-ice-river-land-ice model and the
  CIME case-control system,
- both the production EAMv3 (Fortran) and the next-generation EAMxx /
  SCREAM (C++/Kokkos) atmospheres,
- supported DOE machines (Frontier at OLCF, Perlmutter at NERSC, Aurora at
  ALCF, Chrysalis at LCRC, Compy at PNNL, ...),
- the GitHub-based development workflow used by the E3SM Project.

The skill captures **procedural knowledge** that is normally only acquired
by working alongside an experienced E3SM scientist: which compset to pick
when you only changed one component, how to interpret a 200-line
`e3sm.log.*` traceback, why your restart is not bit-for-bit, and how to
land a feature branch through the `e3sm_integration` test suite.

**Progressive disclosure:**
- `SKILL.md`, routing hub: decision tree, repo layout, quick start, critical rules
- `reference/*.md`, deep-dive docs loaded on demand

## Contents

| Document | What's inside |
|----------|---------------|
| `SKILL.md` | Entry point. Decision tree, repo layout, quick start, critical rules |
| `reference/getting-started.md` | Clone, externals via git submodules, supported DOE machines, build prerequisites |
| `reference/architecture.md` | Components: EAM, EAMxx, ELM, MPAS-Ocean, MPAS-Seaice, MOSART, MALI; driver-mct vs driver-moab |
| `reference/running-cases.md` | `create_newcase`, compsets (WCYCL/F/I/G/B/D), supported resolutions, `case.setup`, `case.build`, `case.submit` |
| `reference/eam-atmosphere.md` | EAMv3 physics (HOMME, ZM, P3, CLUBB, RRTMG, MAM), EAMxx/SCREAM (Kokkos, RRTMGP, SHOC) |
| `reference/mpas-ocean-seaice.md` | Voronoi mesh, time stepping, MARBL biogeochemistry, Icepack physics |
| `reference/output-and-postprocess.md` | h0/h1/h2/h3 history streams, monthly/daily/hourly outputs, e3sm_diags, mpas_analysis, zppy |
| `reference/performance-and-scaling.md` | PE layouts, threading, GPU support, Frontier/Perlmutter/Aurora notes |
| `reference/debugging.md` | Build failures, runtime crashes, conservation, restart bit-for-bit, BFB testing |
| `reference/contributing-pr.md` | PR workflow, integration tests, NextGen vs maintenance branches |

## Sources

This skill is grounded in:

1. **E3SM-Project/E3SM** repository: `README.md`, `AGENTS.md`,
   `CONTRIBUTING.md`, `CITATION.cff`, `mkdocs.yaml`, `run_e3sm.template.sh`,
   the in-tree `docs/` (MkDocs source for https://docs.e3sm.org/E3SM), and
   per-component `components/*/docs/{user-guide,tech-guide,dev-guide}/`.
2. **E3SM-Project/CIME** submodule under `cime/`: case control system,
   `create_newcase`, test driver.
3. **Public documentation:** https://docs.e3sm.org, https://e3sm.org,
   https://github.com/E3SM-Project/E3SM/wiki
4. **Description papers:** Golaz et al. 2019 (E3SMv1, JAMES), Golaz et
   al. 2022 (E3SMv2, JAMES), Caldwell et al. 2021 (SCREAM, JAMES),
   Petersen et al. 2019 (MPAS-Ocean, JAMES), Turner et al. 2022
   (MPAS-Seaice, GMD).

## Install

This skill follows the same layout as
[laps-skill](https://github.com/huangzesen/laps-skill) and the noahmp-skill
sibling:

```
e3sm-skill/
├── SKILL.md              ← routing hub (read first)
├── README.md             ← this file
├── LICENSE               ← MIT
├── .gitignore
└── reference/            ← deep-dive docs
```

To use with a Claude Code or LingTai agent, drop the directory into your
skills library and refresh.

## License

MIT. E3SM itself is governed by the BSD 3-clause license at
https://github.com/E3SM-Project/E3SM/blob/master/LICENSE.
