# Vertical Coordinate and ALE

The single most distinctive design choice in MOM6 is its **generalized
vertical coordinate** driven by an Arbitrary Lagrangian-Eulerian (ALE)
remapping engine. The dynamical core integrates the layer-integrated
primitive equations on a stack of finite-volume layers whose interfaces
are free to move with the flow. After the dynamical step, an **ALE step**
optionally regrids those layers back toward a target coordinate (z*,
sigma, isopycnal, hybrid) and conservatively remaps the state to the new
grid. Switching coordinate systems is a parameter file change, not a
code change.

This page maps the engine.

---

## The ALE step in one paragraph

The Lagrangian half (the dynamical core) advances each layer's volume,
momentum, and tracers under the assumption that the layer interfaces move
with the resolved vertical velocity. After this step, the interfaces have
drifted away from any preferred coordinate. The ALE half then (1)
**regrids**: chooses a new set of target interface positions that satisfy
the user's preferred vertical coordinate (`MOM_regridding`), and (2)
**remaps**: integrates the existing layer profile against the new
interfaces using a high-order conservative reconstruction
(`MOM_remapping`). When the user wants a fully Lagrangian (purely
isopycnal) configuration, the regrid step is a no-op.

```
Lagrangian dynamics (h moves freely)
                            |
                            v
    +------------------------------------------------+
    | ALE step (optional, controlled by USE_REGRIDDING) |
    |                                                |
    |  1. MOM_regridding  ->  pick new target h(*)   |
    |  2. MOM_remapping   ->  remap u, v, T, S, ...  |
    +------------------------------------------------+
                            |
                            v
                  ready for next dynamics step
```

Reference: Adcroft et al. 2019 JAMES (doi:10.1029/2019MS001726) and
Griffies, Adcroft, Hallberg 2020 JAMES.

---

## Coordinate choices

`src/ALE/coord_*.F90` implements the per-coordinate target-grid generator.
The selection is `REGRIDDING_COORDINATE_MODE` in `MOM_input`.

| Mode string | File | Description |
|-------------|------|-------------|
| `Z*` | `coord_zlike.F90` | Quasi-Eulerian z*. Like z but rescaled to swallow free-surface motion |
| `ZSTAR` | `coord_zlike.F90` | Same as Z* (alias) |
| `SIGMA` | `coord_sigma.F90` | Terrain-following. Layers proportional to local depth |
| `RHO` | `coord_rho.F90` | Isopycnal target. Each interface tracks a chosen reference density |
| `HYCOM1` | `coord_hycom.F90` | HYCOM-style hybrid: pressure near the surface, isopycnal in the interior |
| `SIGMA_SHELF_ZSTAR` | `coord_zlike.F90` (composite) | Sigma over shelves, z* off shelves |
| `ADAPTIVE` | `coord_adapt.F90` | Adaptive coordinate that responds to local stratification |

Each coordinate exposes parameters in `MOM_input`. For example:

```
REGRIDDING_COORDINATE_MODE = "Z*"
ALE_COORDINATE_CONFIG = "FILE:vgrid.nc,interfaces=zw"
NK = 75
MIN_THICKNESS = 1.0e-3                ! [m] Floor on layer thickness
```

Or for HYCOM-style hybrid:

```
REGRIDDING_COORDINATE_MODE = "HYCOM1"
ALE_COORDINATE_CONFIG = "HYBRID:dz_init.nc,sigma2,interfaces=salt"
NK = 75
HYBGEN_N_ITERATIONS = 2
```

---

## `MOM_ALE.F90` (top-level driver)

`MOM_ALE_main` is called once per baroclinic step from `MOM.F90`. It:

1. Captures the pre-regrid state through `ALE_PCM_remap_predicted_thicknesses`
   (used to construct an arrival grid that respects the predicted
   continuity step).
2. Calls `regridding_main` (in `MOM_regridding`) to compute the new
   interface positions `h_new`.
3. Calls `remapping_main` (in `MOM_remapping`) to project `u`, `v`, `h`,
   `T`, `S`, and every registered tracer onto the new layers.
4. Optionally calls `pre_remap_diagnostics` and `post_remap_diagnostics`
   so that the diagnostic mediator captures both pre- and post-remap
   versions of selected fields.

`USE_REGRIDDING = True` activates the engine. Setting it `False` reverts
MOM6 to a fully Lagrangian (isopycnal) layered model.

---

## `MOM_regridding.F90`

Picks the new target grid given the current state. Most of the file is
boilerplate that delegates to one of the `coord_*` modules:

```fortran
select case (CS%regridding_scheme)
case (REGRIDDING_ZSTAR)
   call build_zstar_grid(CS%zlike_CS, G, GV, h, dzInterface, ...)
case (REGRIDDING_SIGMA)
   call build_sigma_grid(CS%sigma_CS, G, GV, h, dzInterface, ...)
case (REGRIDDING_RHO)
   call build_rho_grid(CS%rho_CS, G, GV, h, tv, dzInterface, ...)
case (REGRIDDING_HYBGEN)
   call hybgen_regrid(CS%hybgen_CS, G, GV, h, tv, dzInterface, ...)
...
```

Important knobs:

- `MIN_THICKNESS`: floor on layer thickness; protects against zero-volume
  cells.
- `MAX_DEPTH`, `BATHYMETRY_AT_VEL`: how the target grid handles topography.
- `REGRID_COMPRESSIBILITY_FRACTION`: trades accuracy against numerical
  stability for the rho mode.

`MOM_hybgen_regrid.F90`, `MOM_hybgen_remap.F90`, and
`MOM_hybgen_unmix.F90` provide a separate, three-step path used by HYCOM
hybrid: regrid to layered hybrid, remap, then **unmix** dense water that
has accidentally accumulated above its target density layer.

---

## `MOM_remapping.F90`

Conservatively projects the column profile onto the new grid. The chosen
reconstruction determines accuracy at layer interfaces.

`REMAPPING_SCHEME` selects the reconstruction. Names follow the pattern
`<order>_<limiter>`:

| Scheme | File | Notes |
|--------|------|-------|
| `PCM` | `Recon1d_PCM.F90`, `PCM_functions.F90` | Piecewise Constant Method. First-order |
| `PLM` | `Recon1d_PLM_*.F90`, `PLM_functions.F90` | Piecewise Linear with assorted slope limiters |
| `PLM_HYBGEN` | `Recon1d_PLM_hybgen.F90` | PLM matching the HYCOM hybrid limiter |
| `PPM_H4` (default) | `Recon1d_PPM_*.F90`, `PPM_functions.F90` | Piecewise Parabolic, fourth-order edge values |
| `PPM_HYBGEN` | `Recon1d_PPM_hybgen.F90` | PPM with HYCOM hybrid limiting |
| `PPM_CW`, `PPM_CWK` | `Recon1d_PPM_CW.F90`, `Recon1d_PPM_CWK.F90` | Colella-Woodward variants |
| `PQM_IH4IH3` etc. | `Recon1d_*`, `PQM_functions.F90` | Piecewise Quartic, used in research configurations |
| `EMPLM`, `EPPM` | `Recon1d_EMPLM_*.F90`, `Recon1d_EPPM_CWK.F90` | Energetically constrained variants |

Selection examples:

```
REMAPPING_SCHEME = "PPM_H4"           # default
EDGE_VALUE_SCHEME = "PPM_H4"          # how interface edge values are computed
INTERPOLATION_SCHEME = "PLM"          # only relevant if REMAPPING uses interp
```

Reconstruction polynomials live in `polynomial_functions.F90` and
`regrid_edge_values.F90`. Tridiagonal-style edge-value solves go through
`regrid_solvers.F90`.

---

## Working with the ALE engine

### Add a new vertical coordinate

1. Create `coord_<name>.F90` in `src/ALE/` that exports `init_coord_<name>`
   (allocate the control struct), `set_<name>_params`, and
   `build_<name>_grid` (write `dzInterface` given `h`, `T`, `S`).
2. Add a `REGRIDDING_<NAME>` parameter to the enum at the top of
   `MOM_regridding.F90`, plumb the new branch into
   `regridding_main` and `set_regrid_params`.
3. Plumb the human-readable string to the integer in
   `regridding_scheme_from_string`.
4. Wire your new files into `Makefile.in` (autoconf rebuilds the
   dependency list via `ac/makedep` automatically when `make`
   re-walks the source tree).
5. Add at least one test case under `.testing/tc1` or a new `tc*` so the
   regression suite covers it.

### Add a new reconstruction

1. Create `Recon1d_<name>.F90` with subroutines
   `Recon1d_<name>_reconstruct`, `Recon1d_<name>_init`, etc.
2. Add the new scheme name to the `select case (CS%remapping_scheme)`
   block in `MOM_remapping.F90`.
3. Add the optional `Recon1d_type.F90` interface bindings if your scheme
   exposes higher-order edge gradients.
4. Add coverage in `.testing/tc*` MOM_input files
   (`REMAPPING_SCHEME = "<NAME>"`).

### Choose a coordinate at runtime

The single most important parameter is `REGRIDDING_COORDINATE_MODE`. The
common production choices are:

- **`Z*`**: GFDL CM4, OM4 quarter-degree global. Most stable, easiest to
  understand. Good for general circulation work.
- **`HYCOM1`**: GFDL high-resolution coupled climate. Better watermass
  representation than z* in the deep ocean, less sensitive to spurious
  diapycnal mixing.
- **`SIGMA`**: regional shelf models, ROMS-like configurations.
- **`RHO`**: research configurations that require strict isopycnal
  layering. Set `USE_REGRIDDING = False` for fully Lagrangian
  isopycnal mode (the original Hallberg Isopycnal Model behavior).

---

## Diagnostics in coordinate space

The same ALE machinery powers the **diagnostic** Z-, rho-, and sigma-space
output through `MOM_diag_remap.F90`. Diagnostic remapping is a
read-only application of the same remap kernel; see
`output-and-diagnostics.md` for how to ask for `temp_z` or `salt_rho` in
your `diag_table`.

---

## Where to next

- Architecture overview: `architecture.md`
- Sub-grid mixing schemes: `physics-parameterizations.md`
- Diagnostic Z-space output: `output-and-diagnostics.md`
- Adcroft et al. 2019 description paper: doi:10.1029/2019MS001726
