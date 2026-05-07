---
name: mom6
description: >
  Self-contained guide to MOM6 (Modular Ocean Model, version 6), the
  next-generation ocean model developed by NOAA-GFDL and a multi-institution
  consortium. Covers the Arbitrary Lagrangian-Eulerian (ALE) vertical
  coordinate engine, the dynamical core, vertical and lateral
  parameterizations, the FMS-based runtime, the MOM_input/MOM_override
  parameter system, the diag_manager output workflow, ocean-only and coupled
  drivers (FMS_cap, NUOPC, solo_driver), and the regression-based contribution
  workflow. Progressive disclosure: start in this file for routing, drill into
  reference/ for depth.
version: 0.1.0
tags:
  - earth-science
  - ocean-model
  - mom6
  - climate
  - fortran
  - fms
  - ale
  - gfdl
  - cesm
---

# MOM6 (Modular Ocean Model 6) Complete Guide

> **MOM6** = Modular Ocean Model, version 6
> Maintainer: NOAA-GFDL and consortium (Princeton, NCAR, NOAA-EMC, DOE-E3SM)
> Source: https://github.com/NOAA-GFDL/MOM6
> User docs: https://mom6.readthedocs.io
> Examples: https://github.com/NOAA-GFDL/MOM6-examples (and its wiki)
> Description paper: Adcroft et al. 2019, JAMES 11:3167-3211, doi:10.1029/2019MS001726

**What MOM6 does:** Solves the hydrostatic primitive equations for the ocean
on an Arakawa C-grid using a layer-integrated vector-invariant formulation.
Distinguishing feature: a generalized vertical coordinate driven by an
**Arbitrary Lagrangian-Eulerian (ALE)** remapping engine. The same executable
can run in z*, sigma, terrain-following, isopycnal, or HYCOM-style hybrid
modes by changing one parameter file. Ships in many systems including
NOAA-GFDL CM4/OM4/SPEAR, NCAR CESM2/3, DOE E3SM, NOAA UFS, and NOAA SHiELD.

**Who this skill is for:** Agents, students, and ocean modelers who want to
build, run, modify, debug, couple, and contribute to MOM6.

---

## Quick Decision Tree

```
"What do I need?"
│
├─ First time. What is MOM6 and how do I build it?
│  └─ Read: reference/getting-started.md
│     (clone, FMS dependency, autoconf path, mkmf/list_paths path,
│      ocean-only smoke test)
│
├─ I want to understand how the source tree is laid out
│  └─ Read: reference/architecture.md
│     (src/{ALE, core, framework, parameterizations, tracer, ...},
│      config_src/{drivers, infra, memory, external}, pkg/, ac/)
│
├─ I want to understand or change the vertical coordinate
│  └─ Read: reference/vertical-coordinate.md
│     (z*, sigma, isopycnal rho, hybrid HYCOM, ALE remap,
│      MOM_regridding, MOM_remapping, coord_*.F90)
│
├─ I want to change a physics parameterization
│  └─ Read: reference/physics-parameterizations.md
│     (KPP via CVMix, ePBL, internal-tide mixing, kappa-shear,
│      Gent-McWilliams thickness diffusion, MEKE, neutral diffusion,
│      Zanna-Bolton GME)
│
├─ I want to set up and run an experiment
│  └─ Read: reference/running-experiments.md
│     (config_src drivers, MOM_input + MOM_override, input.nml,
│      .testing tc1-tc4 layouts, MOM6-examples wiki)
│
├─ I want to add or change diagnostic output
│  └─ Read: reference/output-and-diagnostics.md
│     (FMS diag_manager, diag_table grammar, native vs _z/_rho/_sigma
│      remapped diagnostics, ocean.stats, chksum_diag)
│
├─ I want to couple MOM6 to an atmosphere or sea-ice model
│  └─ Read: reference/coupling.md
│     (FMS_cap vs NUOPC cap vs solo_driver, ESMF, CESM/E3SM/SHiELD,
│      flux_exchange, data_table for data atmospheres)
│
├─ Model crashes, NaN, CFL error, wrong restart
│  └─ Read: reference/debugging.md
│     (CHECK_BAD_SURFACE_VALS, MOM_PointAccel, debug FCFLAGS,
│      ocean.stats divergence, salt/heat conservation,
│      bit-for-bit restart tests)
│
└─ I want to submit a pull request to NOAA-GFDL/MOM6
   └─ Read: reference/contributing-pr.md
      (fork, dev/gfdl branch, .testing regression suite,
       MOM_parameter_doc churn, code style, Doxygen, GFDL GitLab CI)
```

---

## What's Inside MOM6

```
MOM6/
├── ac/                                  # Autoconf build (top-level configure lives here)
│   ├── configure.ac · Makefile.in · m4/
│   ├── deps/                            # mkmf + FMS fetcher and builder
│   └── makedep                          # Python dependency walker for Fortran sources
├── config_src/
│   ├── drivers/
│   │   ├── solo_driver/                 # Ocean-only executable (no atmos, no ice)
│   │   ├── FMS_cap/                     # GFDL coupler (CM4, OM4, SHiELD)
│   │   ├── nuopc_cap/                   # NUOPC/ESMF cap (CESM, UFS, E3SM)
│   │   ├── ice_solo_driver/             # Standalone ice-shelf driver
│   │   ├── unit_tests/ · timing_tests/  # Test harnesses
│   │   └── STALE_mct_cap/               # Legacy MCT cap (deprecated)
│   ├── external/                        # Null APIs for ODA, BGC, MARBL, drifters
│   ├── infra/{FMS1,FMS2}/               # Thin wrappers around two FMS API generations
│   └── memory/{dynamic_symmetric,dynamic_nonsymmetric}/  # Compile-time memory layout
├── docs/                                # Sphinx + Doxygen docs (mom6.readthedocs.io)
├── pkg/
│   ├── CVMix-src/                       # Vertical mixing community library (KPP, double diff)
│   └── GSW-Fortran/                     # TEOS-10 Gibbs SeaWater toolbox
├── src/
│   ├── ALE/                             # Generalized vertical coordinate engine
│   │   ├── MOM_ALE.F90                  # Top-level ALE driver
│   │   ├── MOM_regridding.F90           # Picks new target grid
│   │   ├── MOM_remapping.F90            # Conservatively remaps state to new grid
│   │   ├── coord_{zlike,sigma,rho,hycom,adapt}.F90
│   │   ├── Recon1d_*.F90                # PCM/PLM/PPM/PQM reconstructions
│   │   └── MOM_hybgen_{regrid,remap,unmix}.F90  # HYCOM-style hybrid
│   ├── core/                            # Dynamical core
│   │   ├── MOM.F90                      # Top-level model driver, holds the state
│   │   ├── MOM_dynamics_split_RK2.F90   # Split barotropic-baroclinic RK2 (default)
│   │   ├── MOM_dynamics_unsplit{,_RK2}.F90
│   │   ├── MOM_barotropic.F90           # Fast (free surface) sub-cycle
│   │   ├── MOM_continuity_PPM.F90       # Layer-thickness PPM continuity
│   │   ├── MOM_CoriolisAdv.F90          # Vector-invariant Coriolis + advection
│   │   ├── MOM_PressureForce_{FV,Montgomery}.F90
│   │   ├── MOM_open_boundary.F90        # OBCs (Flather, Orlanski, ...)
│   │   ├── MOM_porous_barriers.F90      # Sub-grid topography
│   │   ├── MOM_grid.F90 · MOM_verticalGrid.F90
│   │   ├── MOM_variables.F90            # Master state derived types
│   │   └── MOM_forcing_type.F90         # Surface-flux derived type
│   ├── diagnostics/                     # MLD, KdWork, harmonic analysis, sum_output
│   ├── equation_of_state/
│   │   ├── MOM_EOS.F90                  # Dispatch layer
│   │   ├── MOM_EOS_{Wright,Wright_full,Wright_red}.F90
│   │   ├── MOM_EOS_{Jackett06,Roquet_rho,Roquet_SpV,UNESCO,linear}.F90
│   │   ├── MOM_EOS_TEOS10.F90 + TEOS10/  # Wraps pkg/GSW-Fortran
│   │   └── MOM_TFreeze.F90              # Freezing temperature
│   ├── framework/                       # FMS-agnostic infrastructure
│   │   ├── MOM_diag_mediator.F90        # Diagnostic registry, calls FMS diag_manager
│   │   ├── MOM_diag_remap.F90           # Z, rho, sigma remapped diagnostics
│   │   ├── MOM_file_parser.F90          # MOM_input / MOM_override scanner
│   │   ├── MOM_restart.F90 · MOM_io.F90 · MOM_netcdf.F90
│   │   ├── MOM_domains.F90 · MOM_coms.F90 · MOM_hor_index.F90
│   │   ├── MOM_unit_scaling.F90         # Dimensional rescaling for testing
│   │   └── MOM_ANN.F90                  # Neural-network plumbing (Zanna-Bolton)
│   ├── ice_shelf/                       # Ice-shelf coupling and dynamics
│   ├── initialization/                  # Grid, coord, and state initialization
│   ├── parameterizations/
│   │   ├── lateral/                     # GM/MEKE, hor_visc, internal_tides, tidal_forcing,
│   │   │                                # mixed_layer_restrat, Zanna_Bolton GME, SAL
│   │   └── vertical/                    # KPP, ePBL, kappa_shear, tidal_mixing, BBL,
│   │                                    # bulk_mixed_layer, opacity, geothermal, sponges,
│   │                                    # CVMix_{KPP,conv,ddiff,shear}, diabatic_driver
│   ├── tracer/                          # Advection (PPM), neutral_diffusion, hor_bnd_diffusion,
│   │                                    # CFCs, ideal_age, MARBL, OCMIP2_CFC, offline_main
│   └── user/                            # Per-experiment initial conditions and forcing
└── .testing/                            # Local CI: tc0..tc4 + Makefile-based regression suite
```

**For most users:** edit `MOM_input` + `MOM_override` and `diag_table`, then
launch the executable built from `config_src/drivers/<your driver>/`. Source
edits live in `src/` (physics, dynamics, framework) and
`config_src/drivers/` (coupling layer).

---

## Critical Rules

1. **MOM6 always needs FMS.** The model is built on top of NOAA-GFDL's
   Flexible Modeling System. The build either fetches a pinned FMS via
   `make -C ac/deps` or expects an external FMS install findable via
   `FCFLAGS`/`LDFLAGS`. There is no FMS-free build.

2. **Update submodules, every time.**
   ```bash
   git submodule update --init --recursive
   ```
   `pkg/CVMix-src` and `pkg/GSW-Fortran` are vendored as submodules and the
   model will not link without them.

3. **`MOM_input` is the baseline, `MOM_override` is the diff.** The override
   file is parsed second and may use the `#override PARAM = value` directive
   to replace a baseline value. Putting tweaks in `MOM_override` keeps
   experiment diffs small and makes it trivial to compare configurations.

4. **Use the runtime parameter doc files.** Every run writes
   `MOM_parameter_doc.all`, `MOM_parameter_doc.short`, and
   `MOM_parameter_doc.layout` to the run directory. Treat these as the
   authoritative reference for what your run actually used. Never hand-edit a
   `MOM_parameter_doc.*` file.

5. **Pick exactly one memory layout at compile time.** Build with either
   `config_src/memory/dynamic_symmetric/` (default, recommended) or
   `dynamic_nonsymmetric/`, never both. Symmetric grids put velocity points on
   the boundaries; nonsymmetric grids do not and cannot use open boundary
   conditions.

6. **All real arithmetic is reproducible by design.** When editing the
   dynamical core, follow the parenthesization and `reproducing_sum`
   discipline from the code style guide. Sums must be MPI-reproducible. Do
   not introduce `sum()`, `matmul()`, or unparenthesized chained additions.

7. **Default real kind is 8-byte.** `--disable-real-8` exists but is not
   supported for production. Most algorithms assume double precision.

8. **Run experiments outside the source tree.** Each experiment expects its
   own directory containing `MOM_input`, `MOM_override`, `input.nml`,
   `diag_table`, an `INPUT/` subdir for restarts and forcing, and a
   `RESTART/` subdir for output restarts. The MOM6-examples repo and
   `.testing/tc{1,2,3,4}/` show the canonical layout.

9. **`./configure` lives under `ac/`, but you should run it from a separate
   build directory.** The recommended pattern is `mkdir build && cd build &&
   ../ac/configure && make -j`. Out-of-tree builds keep object files away
   from the source.

10. **Coupled builds are not autoconf.** `ac/` only builds the ocean-only
    executable. CESM, E3SM, UFS, and CM4/OM4 builds are driven by the host
    coupled-system build system (CIME, CESM Buildnml, FRE, etc.). Do not try
    to use autoconf for coupled work.

---

## Quick Start (ocean-only, autoconf path)

```bash
# 1. Get the code with submodules (CVMix and GSW are submodules)
git clone --recurse-submodules https://github.com/NOAA-GFDL/MOM6
cd MOM6

# 2. Build the FMS dependency (fetches mkmf + FMS, then compiles)
cd ac/deps
make -j
cd ../..

# 3. Generate ./configure inside ac/
cd ac
autoreconf
cd ..

# 4. Out-of-tree build of the ocean-only executable
mkdir -p build && cd build
../ac/configure                    # autodetects FC, netCDF, MPI
make -j                            # produces ./MOM6 in this directory
cd ..

# 5. Smoke test against tc1 (a low-res benchmark configuration)
cd .testing
make -j                            # builds .testing/build/symmetric/MOM6 etc.
make -j test                       # runs the regression suite
```

Once `make -j test` is green, switch to the
[MOM6-examples wiki](https://github.com/NOAA-GFDL/MOM6-examples/wiki) for
real ocean experiments (double_gyre, OM4_025, ocean_SIS2, NeverWorld,
benchmark, ...) and to `reference/running-experiments.md` for the
MOM_input/MOM_override workflow.

---

## Reference Documents

| Document | What's inside |
|----------|---------------|
| `reference/getting-started.md` | Two-repo layout (MOM6 + MOM6-examples), submodules, FMS dependency, supported compilers, autoconf path, `.testing` smoke test, common build pitfalls |
| `reference/architecture.md` | `src/{ALE,core,framework,parameterizations,tracer,equation_of_state,ice_shelf,user,...}` walkthrough, `config_src/{drivers,infra,memory,external}`, `pkg/`, the `MOM_control_struct` and surface forcing types, dynamical-core call chain |
| `reference/vertical-coordinate.md` | Generalized vertical coordinate, ALE method, `MOM_ALE` / `MOM_regridding` / `MOM_remapping`, `coord_{zlike,sigma,rho,hycom,adapt}.F90`, PCM/PLM/PPM/PQM reconstructions, `Recon1d_*.F90`, HYCOM hybrid (`hybgen`) |
| `reference/physics-parameterizations.md` | Vertical: KPP via CVMix, ePBL, kappa_shear, internal-tide mixing (St Laurent / Polzin / Melet), bulk mixed layer, double diffusion, opacity. Lateral: Smagorinsky/Laplacian/biharmonic viscosity, Gent-McWilliams thickness diffusion, MEKE, mixed-layer restrat (Fox-Kemper / Bodner23), neutral diffusion, GME (Zanna-Bolton) |
| `reference/running-experiments.md` | `config_src` drivers, `MOM_input` baseline + `MOM_override` diff, `input.nml`, `diag_table`, `data_table`, `MOM_parameter_doc.{all,short,layout}`, `.testing/tc1..tc4` layouts, MOM6-examples |
| `reference/output-and-diagnostics.md` | FMS `diag_manager`, `diag_table` grammar, `MOM_diag_mediator`, native vs `_z` / `_rho` / `_sigma` diagnostics, `MOM_diag_remap`, `ocean.stats`, `chksum_diag`, `MOM_sum_output`, restart files |
| `reference/coupling.md` | `solo_driver` for ocean-only, `FMS_cap` for GFDL CM4/OM4/SHiELD, `nuopc_cap` for CESM/UFS/E3SM, `STALE_mct_cap` (deprecated), surface forcing types, `flux_exchange`, `data_table` for data atmospheres, ESMF caps |
| `reference/debugging.md` | Build vs runtime errors, NaN traps with debug FCFLAGS, CFL violations, `MOM_PointAccel`, `CHECK_BAD_SURFACE_VALS`, `ocean.stats` divergence, salt/heat conservation budgets, restart bit-for-bit failures, `DEBUG = True` and `DEBUG_TRUNCATIONS` |
| `reference/contributing-pr.md` | Fork model, `dev/gfdl` target branch, `.testing` regression suite, code style guide, reproducible-sum discipline, Doxygen `!>` / `!<` conventions, MOM_parameter_doc churn, GFDL GitLab CI mirror |
