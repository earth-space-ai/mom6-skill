# MOM6 Architecture

MOM6 is structured around a clear separation between (1) the physics and
numerics that are always compiled (`src/`), (2) the build configuration
that varies with the host system (`config_src/`), and (3) third-party
packages used through symbolic links (`pkg/`). The defining design choice
is the **Arbitrary Lagrangian-Eulerian (ALE)** layered formulation, which
isolates the dynamical core from the choice of vertical coordinate.

This page is the map you need before you edit anything.

---

## Top-level layout

```
MOM6/
|-- ac/                    # Autoconf build (ocean-only)
|-- config_src/            # Drivers, infra wrappers, memory layouts
|-- docs/                  # Sphinx + Doxygen sources for mom6.readthedocs.io
|-- pkg/                   # Third-party submodules (CVMix, GSW)
|-- src/                   # Always-compiled core: physics + numerics + framework
`-- .testing/              # Small regression suite tc0..tc4
```

The compiled object set is always `src/` plus exactly one driver from
`config_src/drivers/`, exactly one memory layout from `config_src/memory/`,
exactly one FMS infra wrapper from `config_src/infra/`, and selected null or
real modules from `config_src/external/`.

---

## `src/` walkthrough

### `src/core/` (dynamical core)

| File | Role |
|------|------|
| `MOM.F90` | Top-level model driver. Holds `MOM_control_struct`, sequences ALE, dynamics, thermodynamics, diagnostics |
| `MOM_dynamics_split_RK2.F90` | Default time integrator: split barotropic/baroclinic RK2 |
| `MOM_dynamics_split_RK2b.F90` | Variant of split RK2 |
| `MOM_dynamics_unsplit.F90`, `MOM_dynamics_unsplit_RK2.F90` | Unsplit modes for testing and short runs |
| `MOM_barotropic.F90` | Fast (free-surface) sub-cycle solved on the barotropic timestep |
| `MOM_continuity_PPM.F90`, `MOM_continuity.F90` | Layer thickness equation (PPM advection of `h`) |
| `MOM_CoriolisAdv.F90` | Vector-invariant Coriolis + kinetic-energy gradient |
| `MOM_PressureForce.F90`, `MOM_PressureForce_FV.F90`, `MOM_PressureForce_Montgomery.F90` | Finite-volume and Montgomery-potential pressure-gradient forms |
| `MOM_open_boundary.F90`, `MOM_boundary_update.F90` | Open boundary conditions (Flather, Orlanski, etc.) |
| `MOM_porous_barriers.F90` | Sub-grid topography porosity |
| `MOM_grid.F90`, `MOM_verticalGrid.F90`, `MOM_dyn_horgrid.F90`, `MOM_transcribe_grid.F90` | Grid metric data structures |
| `MOM_variables.F90` | Master state derived types (thermo, surface, vertvisc, BT_cont) |
| `MOM_forcing_type.F90` | Surface fluxes derived type, shared with all coupling drivers |
| `MOM_density_integrals.F90`, `MOM_interface_heights.F90`, `MOM_isopycnal_slopes.F90` | Diagnostic helpers used by the pressure gradient and lateral mixing |
| `MOM_stoch_eos.F90` | Stochastic equation-of-state perturbation |
| `MOM_check_scaling.F90`, `MOM_checksum_packages.F90` | Dimensional consistency and reproducible-checksum helpers |

### `src/ALE/` (vertical coordinate engine)

The top-level entry is `MOM_ALE.F90`, which drives the regrid + remap cycle
each timestep when ALE is enabled. The two halves are decoupled:

- `MOM_regridding.F90` decides what the new vertical grid should be
  (target interface positions for z*, sigma, rho, hybrid, ...).
- `MOM_remapping.F90` conservatively projects the layer-integrated state
  onto the new grid using a chosen reconstruction.

Coordinate-specific target-grid generators:

```
coord_zlike.F90    # z, z*, depth-following
coord_sigma.F90    # terrain-following
coord_rho.F90      # density (isopycnal target)
coord_hycom.F90    # HYCOM-style layered hybrid
coord_adapt.F90    # Adaptive coordinate
```

Reconstructions used by the remap step (one file per reconstruction
flavor):

```
PCM_functions.F90, PLM_functions.F90, PPM_functions.F90, PQM_functions.F90, P1M_functions.F90, P3M_functions.F90
Recon1d_PCM.F90, Recon1d_PLM_*.F90, Recon1d_PPM_*.F90, Recon1d_EMPLM_*.F90, Recon1d_MPLM_*.F90, Recon1d_EPPM_CWK.F90, Recon1d_PPM_H4_2018.F90, Recon1d_PPM_H4_2019.F90
```

The HYCOM hybrid path has its own three-step regrid/remap/unmix:

```
MOM_hybgen_regrid.F90    # Pick layered hybrid target grid
MOM_hybgen_remap.F90     # Remap to it
MOM_hybgen_unmix.F90     # Unmix dense water trapped above its target layer
```

Common helpers: `regrid_consts.F90`, `regrid_edge_values.F90`,
`regrid_interp.F90`, `regrid_solvers.F90`, `polynomial_functions.F90`.

See `vertical-coordinate.md` for the deep dive.

### `src/parameterizations/`

Two sibling subdirectories plus a CVMix symlink:

```
parameterizations/
|-- CVmix -> ../../pkg/CVMix-src/src/shared/    # symlink into the CVMix submodule
|-- lateral/
|-- vertical/
`-- stochastic/
```

`vertical/` covers boundary layers and interior diapycnal mixing:

| File | Scheme |
|------|--------|
| `MOM_CVMix_KPP.F90` | KPP boundary layer through CVMix |
| `MOM_energetic_PBL.F90` | ePBL (Reichl & Hallberg 2018) |
| `MOM_bulk_mixed_layer.F90` | Bulk mixed layer (isopycnal mode) |
| `MOM_kappa_shear.F90` | Shear-driven mixing (Jackson et al. 2008) |
| `MOM_CVMix_shear.F90`, `MOM_CVMix_conv.F90`, `MOM_CVMix_ddiff.F90` | Other CVMix schemes |
| `MOM_tidal_mixing.F90`, `MOM_internal_tide_input.F90` | St Laurent / Polzin / Melet internal-tide mixing |
| `MOM_diapyc_energy_req.F90` | Energy diagnostics for diapycnal mixing |
| `MOM_set_diffusivity.F90`, `MOM_set_viscosity.F90` | Aggregate diffusivity and viscosity assembly |
| `MOM_diabatic_driver.F90`, `MOM_diabatic_aux.F90` | Top-level diabatic step |
| `MOM_vert_friction.F90` | Implicit vertical viscosity solver |
| `MOM_entrain_diffusive.F90` | Layered-isopycnal diapycnal entrainment (Hallberg 2000) |
| `MOM_geothermal.F90` | Geothermal heat flux at the bottom |
| `MOM_opacity.F90` | Shortwave penetration optical properties |
| `MOM_bkgnd_mixing.F90` | Background interior diffusivity profiles |
| `MOM_full_convection.F90` | Static convective adjustment fallback |
| `MOM_regularize_layers.F90` | Layer thickness regularization (isopycnal mode) |
| `MOM_sponge.F90`, `MOM_ALE_sponge.F90` | Restoring sponges (legacy + ALE-aware) |

`lateral/` covers along-isopycnal and lateral processes:

| File | Scheme |
|------|--------|
| `MOM_hor_visc.F90` | Laplacian + biharmonic horizontal viscosity (linear, Smagorinsky, Leith) |
| `MOM_thickness_diffuse.F90` | Gent-McWilliams thickness diffusion |
| `MOM_lateral_mixing_coeffs.F90` | KhTr / KhTh assembly, resolution function, Visbeck scaling |
| `MOM_MEKE.F90`, `MOM_MEKE_types.F90` | Mesoscale Eddy Kinetic Energy (Jansen, Marshall) |
| `MOM_mixed_layer_restrat.F90` | Mixed-layer restratification (Fox-Kemper 2008, Bodner 2023) |
| `MOM_internal_tides.F90` | Internal-tide energy propagation |
| `MOM_tidal_forcing.F90` | Astronomical tidal body forcing |
| `MOM_self_attr_load.F90`, `MOM_load_love_numbers.F90`, `MOM_spherical_harmonics.F90` | Self-attraction and loading via spherical harmonics |
| `MOM_Zanna_Bolton.F90` | Zanna-Bolton GME closure (and `MOM_ANN.F90` for the neural net version) |
| `MOM_interface_filter.F90`, `MOM_streaming_filter.F90` | Layer-mode filters |
| `MOM_wave_drag.F90` | Bottom internal-wave drag |

See `physics-parameterizations.md` for the deep dive.

### `src/equation_of_state/`

Dispatch through `MOM_EOS.F90`. Available equations of state:

```
MOM_EOS_linear.F90              # Constant alpha and beta
MOM_EOS_Wright.F90              # Wright (1997) (legacy)
MOM_EOS_Wright_full.F90         # Wright with full coefficients
MOM_EOS_Wright_red.F90          # Wright with reduced coefficients
MOM_EOS_UNESCO.F90              # UNESCO 1980 (legacy)
MOM_EOS_Jackett06.F90           # Jackett et al. 2006
MOM_EOS_Roquet_rho.F90          # Roquet et al. (in-situ rho)
MOM_EOS_Roquet_SpV.F90          # Roquet (specific volume)
MOM_EOS_TEOS10.F90 + TEOS10/    # Wraps pkg/GSW-Fortran for TEOS-10
```

Plus `MOM_EOS_base_type.F90` (shared abstract type),
`MOM_temperature_convert.F90` (conservative <-> potential temperature), and
`MOM_TFreeze.F90` (freezing temperature for sea-ice contact).

### `src/tracer/`

Tracer registry, advection, lateral and vertical diffusion, plus
example tracers:

```
MOM_tracer_registry.F90, MOM_tracer_types.F90, MOM_tracer_flow_control.F90
MOM_tracer_advect.F90, MOM_tracer_advect_schemes.F90, MOM_tracer_diabatic.F90
MOM_tracer_hor_diff.F90, MOM_hor_bnd_diffusion.F90, MOM_neutral_diffusion.F90
MOM_offline_main.F90, MOM_offline_aux.F90, MOM_tracer_Z_init.F90
ideal_age_example.F90, oil_tracer.F90, dye_example.F90, dyed_obc_tracer.F90
boundary_impulse_tracer.F90, pseudo_salt_tracer.F90, advection_test_tracer.F90
DOME_tracer.F90, ISOMIP_tracer.F90, RGC_tracer.F90, nw2_tracers.F90
MOM_OCMIP2_CFC.F90, MOM_CFC_cap.F90
MARBL_tracers.F90, MARBL_forcing_mod.F90    # Hooks into the MARBL BGC library
```

### `src/framework/` (FMS-agnostic infrastructure)

The framework layer wraps FMS so that physics modules never call FMS
directly:

| File | Role |
|------|------|
| `MOM_diag_mediator.F90` | Diagnostic registry. Wraps FMS `diag_manager` |
| `MOM_diag_remap.F90`, `MOM_diag_buffers.F90` | Native -> Z / rho / sigma diagnostic remapping |
| `MOM_file_parser.F90` | `MOM_input` / `MOM_override` scanner; emits `MOM_parameter_doc.*` |
| `MOM_get_input.F90` | Bootstraps `input.nml` reading |
| `MOM_io.F90`, `MOM_io_file.F90`, `MOM_netcdf.F90` | I/O abstraction |
| `MOM_restart.F90` | Restart write/read coordination |
| `MOM_domains.F90`, `MOM_coms.F90` | Domain decomposition and communication wrappers |
| `MOM_hor_index.F90`, `MOM_dyn_horgrid.F90` | Horizontal index types |
| `MOM_horizontal_regridding.F90`, `MOM_interpolate.F90` | Online horizontal interpolation |
| `MOM_unit_scaling.F90`, `MOM_unique_scales.F90` | Dimensional rescaling for unit tests |
| `MOM_data_override.F90` | Override fields with data from `data_table` (FMS) |
| `MOM_random.F90`, `MOM_murmur_hash.F90` | Reproducible randomness and hashing |
| `MOM_checksums.F90` | Bit-identical checksums for state |
| `MOM_error_handler.F90`, `MOM_document.F90` | Logging, fatal aborts, parameter logging |
| `MOM_cpu_clock.F90`, `MOM_write_cputime.F90` | Profiling |
| `MOM_array_transform.F90`, `MOM_intrinsic_functions.F90`, `MOM_safe_alloc.F90`, `MOM_string_functions.F90` | Utility kernels |
| `MOM_ANN.F90` | Neural-network feed-forward used by `MOM_Zanna_Bolton` |
| `MOM_coupler_types.F90` | Boundary-flux derived types shared with couplers |
| `MOM_ensemble_manager.F90`, `MOM_unit_tests.F90`, `MOM_unit_testing.F90` | Test harness |
| `posix.F90`, `posix.h` | Process-control helpers (signals, file ops) |
| `version_variable.h`, `MOM_memory_macros.h` | Compile-time tags and memory macros |

### Other `src/` directories

```
src/diagnostics/         # MLD, KdWork, harmonic analysis, sum_output, PointAccel
src/initialization/      # Grid, vertical-coord, and 3D state initialization
src/ice_shelf/           # Ice-shelf interaction and sub-shelf circulation
src/ocean_data_assim/    # Hooks for incremental update DA
src/user/                # Per-experiment ICs and surface forcing modules
```

`src/diagnostics/MOM_PointAccel.F90` is the single most useful debugging
module: it dumps every term in the momentum and tracer budgets at one
column when `U_TRUNC_FILE` or `V_TRUNC_FILE` triggers, or on demand.

---

## `config_src/` walkthrough

`config_src/` is sliced four ways. Each MOM6 build picks one entry from
each axis.

### `config_src/drivers/`

| Driver | Used by |
|--------|---------|
| `solo_driver/` | Ocean-only standalone executable. Built by `ac/configure`. Contains `MOM_driver.F90`, `MOM_surface_forcing.F90` |
| `FMS_cap/` | GFDL coupler. Used by CM4, OM4, SHiELD. Files: `ocean_model_MOM.F90`, `MOM_surface_forcing_gfdl.F90` |
| `nuopc_cap/` | NUOPC/ESMF cap. Used by CESM, NOAA UFS, DOE E3SM. Files: `ocn_comp_NUOPC.F90`, `mom_cap.F90`, `mom_cap_methods.F90`, `mom_ocean_model_nuopc.F90`, `mom_surface_forcing_nuopc.F90` |
| `ice_solo_driver/` | Standalone ice-shelf driver |
| `STALE_mct_cap/` | Legacy MCT cap, deprecated since CESM moved to NUOPC |
| `unit_tests/` | Built by `make build.unit` in `.testing/` |
| `timing_tests/` | Built by `make build.timing` in `.testing/` |

### `config_src/infra/`

```
infra/FMS1/    # Wrappers for the FMS 2019 API
infra/FMS2/    # Wrappers for FMS 2022+ I/O API (dmsg-style)
```

Pick one. The framework code in `src/framework/` calls these wrappers,
never FMS directly.

### `config_src/memory/`

```
memory/dynamic_symmetric/      # Default. Velocity points on cell faces
memory/dynamic_nonsymmetric/   # Smaller arrays. No OBCs allowed.
```

Static memory layout is also supported (build with `MOM_memory.h` baked in
at compile time) but is rarely used outside production-pinned model
versions.

### `config_src/external/`

Null APIs that satisfy MOM6's external-package linkage when the real
package is absent:

```
external/GFDL_ocean_BGC/    # GFDL biogeochem (BLING / COBALT) hooks
external/MARBL/             # NCAR MARBL BGC hooks
external/ODA_hooks/         # Ocean data assimilation hooks
external/database_comms/    # Database comms (e.g., SmartSim)
external/drifters/          # Lagrangian drifter hooks
external/stochastic_physics/  # Stochastic physics package hooks
```

When you want the real package, you compile its sources alongside `src/`
and skip the matching `external/` null module.

---

## `pkg/`

Two third-party submodules:

- `pkg/CVMix-src` (https://github.com/CVMix/CVMix-src) supplies the KPP and
  associated vertical-mixing kernels through `src/parameterizations/CVmix`
  (a symlink).
- `pkg/GSW-Fortran` (https://github.com/TEOS-10/GSW-Fortran) supplies the
  TEOS-10 Gibbs SeaWater toolbox through `src/equation_of_state/TEOS10`
  (a symlink).

Both are kept at pinned commits in `.gitmodules`.

---

## The dynamical-core call chain

Per baroclinic timestep, with split RK2 (the default), `MOM.F90` runs:

```
step_MOM
|-- ALE_main                           (regrid + remap state to a fresh grid)
|-- thickness_diffuse                  (GM, MEKE, mixed-layer restrat)
|-- step_MOM_dynamics_split_RK2
|   |-- continuity_PPM (predictor)     (advect h)
|   |-- CoriolisAdv                    (vector-invariant Coriolis + KE grad)
|   |-- PressureForce                  (FV or Montgomery)
|   |-- horizontal_viscosity (predictor)
|   |-- barotropic_step                (sub-cycle the free-surface mode)
|   |-- continuity_PPM (corrector)
|   |-- vertvisc_remnant + vertvisc    (implicit vertical friction)
|-- diabatic                           (KPP/ePBL + interior diffusivity)
|   |-- set_diffusivity + set_viscosity
|   |-- KPP_calculate or ePBL_column_main
|   |-- tidal_mixing
|   |-- bulk_mixed_layer (if isopycnal)
|   |-- tracer vertical diffusion
|-- advect_tracer                      (PPM tracer advection)
|-- tracer_hor_diff                    (Redi neutral diffusion)
`-- write_diagnostics                  (FMS diag_manager flush)
```

`MOM_dynamics_split_RK2.F90` is the place to start when reading the core.
The split mode runs the slow baroclinic equations with one large `dt`
(usually 900-3600 s) and sub-cycles the fast barotropic mode with many
small steps (50-200 per baroclinic step).

---

## Master state derived types

`src/core/MOM_variables.F90` declares the master derived types passed
through the dynamics:

- `thermo_var_ptrs` (T, S, salinity-restoring fields, tracer registry)
- `surface` (sea surface T, S, height, mixed-layer depth, currents)
- `vertvisc_type` (vertical viscosity and diffusivity arrays)
- `BT_cont_type` (barotropic continuity coupling fields)
- `cont_diag_ptrs`, `accel_diag_ptrs` (diagnostic pointer bundles)

`src/core/MOM_forcing_type.F90` declares `forcing` (momentum, heat,
freshwater fluxes; shared with every coupling driver).

`src/core/MOM.F90` declares `MOM_control_struct`, the per-instance handle
that holds pointers to every other registered control struct (ALE,
diabatic, dynamics, tracer registry, diag mediator, ...).

When adding a new state variable: add the field to the appropriate type in
`MOM_variables.F90`, allocate it in the relevant `*_init` routine,
register it for diagnostics in the right `register_*_diags` routine, and
add it to restart with `register_restart_field`.

---

## Where to next

- Vertical-grid deep dive: `vertical-coordinate.md`
- Physics parameterizations deep dive: `physics-parameterizations.md`
- Set up an experiment: `running-experiments.md`
- Diagnostics and remapping: `output-and-diagnostics.md`
