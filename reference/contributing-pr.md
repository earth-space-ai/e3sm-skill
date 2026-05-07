# Contributing to E3SM via Pull Request

E3SM is developed openly on GitHub at
https://github.com/E3SM-Project/E3SM. Bug fixes, new physics, and
infrastructure changes flow through pull requests reviewed by component
leads and integrated by the E3SM Integration Team. This guide walks
through the standard contribution flow.

Project guidance:
https://e3sm.org/model/running-e3sm/developing-e3sm/

CONTRIBUTING file in repo: `CONTRIBUTING.md`

---

## Before you start

1. **Check the science plan.** New features that are *not* part of an
   E3SM-funded science task may not be accepted. The CONTRIBUTING file
   states explicitly: "We may not have the resources to test/evaluate new
   or changed features from non-staff. Your feature PR will get attention
   if it's part of the E3SM science plan and coordinated by its
   management."
2. **Check existing issues and discussions.** Search
   [Issues](https://github.com/E3SM-Project/E3SM/issues) and
   [Discussions](https://github.com/E3SM-Project/E3SM/discussions) for
   prior work.
3. **Open an issue for any bug.** Include the failing case, machine,
   compiler, compiler version, and a minimal reproducer. Bugs in
   external code (FATES, MARBL, Icepack, MCT, scorpio) should also be
   reported in the upstream repo.

---

## Step 1: Fork the repo

Visit https://github.com/E3SM-Project/E3SM → click **Fork**.

Then clone your fork with submodules:

```bash
git clone --recursive git@github.com:<your_user>/E3SM.git
cd E3SM
git remote add upstream git@github.com:E3SM-Project/E3SM.git
git remote -v
# → origin   = your fork
# → upstream = E3SM-Project/E3SM
```

Sync regularly:

```bash
git fetch upstream
git checkout master
git merge upstream/master
git push origin master
```

---

## Step 2: Branch from `master`

E3SM follows a "git workflows" model (per `AGENTS.md`) but without a
`seen` branch. The mainline integration branch is `master`. Every
feature branches off `master` and merges back into `master` through a PR.

**Branch naming convention** (enforced):

```
<github_username>/<source_code_area_or_component>/<feature_description>
```

Examples:

```
kwu/eam/zm-trigger-improvement
kwu/eamxx/p3-rime-bugfix
kwu/elm/snow-albedo-update
kwu/cime/perlmutter-batch-fix
kwu/docs/getting-started-cleanup
```

```bash
git checkout -b kwu/eam/zm-trigger-improvement upstream/master
```

PRs with branches that do not match this pattern get flagged.

---

## Step 3: Implement and test

Make atomic, granular commits. One logical change per commit.

```bash
git status
git add components/eam/src/physics/cam/zm_conv.F90
git commit -m "ZM: add upper-troposphere CAPE limit"
```

For each change, run a scoped test (see `running-cases.md` for the
"AGENTS test compset" matrix):

| Touched | Compset | Resolution |
|---------|---------|------------|
| ELM | `I1850CNPRDCTCBCTOP` | `ne4pg2_ne4pg2` |
| EAM | `F1850` | `ne4pg2_oQU480` |
| EAMxx | `F2010-SCREAMv1` | `ne4pg2_ne4pg2` |
| MOSART | `RMOSGPCC` | `r05_r05` |
| MPAS-Ocean | `CMPASO-NYF` | `T62_oQU120` |
| MPAS-Seaice | `DTESTM` | `T62_oQU240` |
| 2+ components | `WCYCL1850NS` | `ne4pg2_r05_oQU480` |

```bash
cd cime/scripts
./create_newcase --case ../../cases/test_zm \
    --compset F1850 --res ne4pg2_oQU480 --mach <MACHINE>
cd ../../cases/test_zm
./case.setup
./case.build
./xmlchange STOP_OPTION=ndays,STOP_N=5
./case.submit
```

For EAMxx, also run the standalone tests:

```bash
cd components/eamxx
./scripts/test-all-eamxx -m <MACHINE>
```

---

## Step 4: Run the appropriate CIME test suite

CIME has a battery of integration tests. The most important suites:

| Suite | Scope |
|-------|-------|
| `e3sm_developer` | Everyday developer regression suite |
| `e3sm_integration` | Pre-merge integration testing |
| `e3sm_atm_developer` | Atmosphere-only |
| `e3sm_land_developer` | ELM-only |
| `e3sm_mosart_developer` | MOSART-only |
| `e3sm_land_exeshare` | ELM with shared executable |

```bash
cd cime/scripts
./create_test e3sm_developer --machine <MACHINE> --baseline-dir <BASELINE_DIR>
```

Each test's status appears in its test directory:

```
cd <test_dir>
cat TestStatus
tail TestStatus.log
```

The full list of test suites lives in `cime_config/tests.py`. Common
test types:

| Type | Meaning |
|------|---------|
| `SMS` | Short smoke test (just verify it runs) |
| `SMS_D` | Smoke test with `DEBUG=TRUE` |
| `ERS` | Exact restart test (must be bit-for-bit) |
| `ERS_Ld` | Exact restart test for d days |
| `PEM` | Bit-for-bit when changing MPI task count |
| `PET` | Bit-for-bit when changing thread count |

Test naming: `TEST_TYPE.RESOLUTION.COMPSET[.MACHINE][.TESTMOD]`

Example: `SMS_D_Ln5_P4.ne4pg2_oQU480.F2010.ghci-oci_gnu`

---

## Step 5: Push and open the PR

```bash
git push origin kwu/eam/zm-trigger-improvement
```

On the GitHub UI of your fork, click "Compare & pull request":

- **Base:** `E3SM-Project:master`
- **Compare:** `<you>:kwu/eam/zm-trigger-improvement`

### PR description rules (from `AGENTS.md`)

The PR description must have **two parts** separated by a blank line:

1. A brief description in **plain text with no markdown or other
   formatting**.
2. The full description with markdown formatting.

Both should use **imperative tense** starting with a verb like `Fix`,
`Add`, `Update`, `Refactor`, `Remove`.

Bad: "ZM trigger improvements"
Good: "Add upper-troposphere CAPE limit to ZM trigger"

### What to include

- **What changed:** physical/computational summary.
- **Why:** scientific motivation, link to issue, paper reference.
- **How it was tested:** which compsets, which test suites, baseline
  comparisons. Include the test suite name and the
  `my.cdash.org` build that ran it if available.
- **BFB statement:** is this bit-for-bit with `master`? If not, list
  affected compsets and the expected change in solution.
- **Documentation:** link to any updated `docs/` or `components/*/docs/`
  files.

---

## Step 6: Iterate during review

Component leads (and the relevant subgroup) will:

- Inspect physics/numerics correctness
- Run the developer or integration test suite on a CDash machine
- Comment on style and naming
- Request changes if needed

CDash results: https://my.cdash.org/index.php?project=E3SM

GitHub Actions CI runs `eamxx-sa-testing.yml` and `eamxx-v1-testing.yml`
on `ghci-oci` (Oracle Cloud) and `weaver` (SNL). For non-EAMxx PRs,
nightly testing on supported machines drives most of the
go/no-go decision.

Push additional commits to the same branch in the same direction; they
attach to the open PR automatically.

---

## Cosmetic-only changes

Per `CONTRIBUTING.md`:

> Changes that are cosmetic in nature and do not add anything substantial
> to the stability, functionality, or testability of E3SM will generally
> not be accepted from non-staff.

Whitespace-only and pure-formatting PRs from external contributors are
typically declined. Bundle cosmetic cleanups with a substantive change
in the same area.

---

## NextGen vs maintenance branches

E3SM-Project occasionally maintains parallel "NextGen" development
branches alongside `master`. These are typically used for breaking
changes that need to bake in a feature branch before being landed on
`master`. Examples:

- An `omega-integration` branch for integrating the next-generation
  ocean model
- A `nextgen-coupler` branch for driver-moab development
- Component-specific `nextgen-*` branches

When a NextGen branch exists, the component lead will direct PRs to it
explicitly. The default for an unsolicited PR is always `master`.

The previous `acme-climate` historical branch and `release-v*` tagged
release points are read-only.

---

## External submodule changes

If your work changes code in an external repo (FATES, MARBL, Icepack,
mam4xx, scorpio, EKAT, YAKL, MCT, ...), you must:

1. Submit a PR to the **upstream** repo first.
2. Wait for it to be merged upstream.
3. Open an E3SM PR that bumps the submodule SHA in `.gitmodules` (or
   the corresponding fleximod config) to the merged commit.

Single-PR coordinated changes that touch both the upstream and E3SM
require explicit coordination with the integration team.

---

## Common contribution pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| PR title rejected | Not in imperative voice or wrong format | Rewrite as "Fix X" or "Add Y" |
| Branch name flagged | Doesn't match `<user>/<area>/<topic>` | Rename branch and force-push |
| `e3sm_integration` fails after merge | New test added to suite that wasn't run | Run the new test locally before merge |
| BFB violation in `ERS` | Stateful variable not in restart | Add to restart in component IO module |
| BFB violation in `PEM` | Order-dependent global reduction | Use ordered reduction or sort first |
| Submodule SHA bump fails | Upstream PR not actually merged | Wait for upstream merge |
| Reviewer asks "what about EAMxx?" | EAMxx and EAM share concept but not code | Either port the change or note explicitly that EAMxx is unaffected |

---

## License

E3SM is released under the BSD 3-clause license:
https://github.com/E3SM-Project/E3SM/blob/master/LICENSE

By submitting a PR, you certify that your contribution can be released
under that license.

---

## Where to next

- Map the components: `architecture.md`
- Pick the right scoped compset for your test: `running-cases.md`
- Diagnose a test failure on your branch: `debugging.md`
- Tune for the test machine: `performance-and-scaling.md`
