# Physics Parameterizations

Sub-grid scale physics in MOM6 lives under `src/parameterizations/`,
split into `vertical/` (fluxes oriented along the vertical), `lateral/`
(fluxes oriented along the horizontal), and a CVMix symlink into the
`pkg/CVMix-src` submodule. The diabatic driver
(`MOM_diabatic_driver.F90`) sequences the vertical schemes; the
thickness-diffusion driver (`MOM_thickness_diffuse.F90`) sequences the
lateral mesoscale closure; the dynamical core handles momentum
viscosity inline.

This page lists every scheme, the file that implements it, and the
`MOM_input` switch that turns it on.

---

## Vertical: surface boundary layer

Two production-grade boundary-layer schemes ship with MOM6. Pick exactly
one through the `MOM_input` switches below.

### KPP (K-profile parameterization)

`src/parameterizations/vertical/MOM_CVMix_KPP.F90` wraps the CVMix
implementation of the Large, McWilliams, Doney 1994 KPP scheme. The KPP
boundary layer depth is diagnosed from a bulk Richardson criterion; the
non-local transport term is enabled by `KPP%MATCH_TECHNIQUE`.

Switches:

```
USE_KPP = True
KPP%RI_CRIT = 0.3
KPP%INTERP_TYPE2 = "LMD94"
KPP%MATCH_TECHNIQUE = "MatchGradient"
KPP%LT_K_ENHANCEMENT = False     # Langmuir-turbulence enhancement
```

KPP depends on the vertical viscosity returned by `set_viscosity` and
contributes diapycnal diffusivity to `set_diffusivity`. Its diffusivities
combine with all other contributions before the implicit
diabatic step.

### ePBL (Energetic Planetary Boundary Layer)

`src/parameterizations/vertical/MOM_energetic_PBL.F90` implements
Reichl & Hallberg 2018, which constrains the boundary layer
diffusivity by an explicit TKE budget instead of a bulk Richardson
profile. ePBL is the default for many GFDL and CESM-MOM6 production
configurations because it interacts cleanly with the ALE step.

Switches:

```
ENERGETICS_SFC_PBL = True
EPBL_USTAR_MIN = 1.0e-7
USE_LA_LI2016 = True              # Langmuir scaling (Li et al. 2016)
EPBL_LANGMUIR_SCHEME = "ADDITIVE"
TKE_DECAY = 2.5
MIX_LEN_EXPONENT = 1.0
```

ePBL and KPP are mutually exclusive in production. The bulk-mixed-layer
scheme below is the third option, used only with the layered isopycnal
mode.

### Bulk mixed layer (BML)

`src/parameterizations/vertical/MOM_bulk_mixed_layer.F90` implements the
two-layer bulk mixed layer used with the **isopycnal** vertical mode
(`USE_REGRIDDING = False`). Not used in ALE mode.

```
BULKMIXEDLAYER = True             # Only with isopycnal mode
```

---

## Vertical: interior and bottom-driven mixing

Several schemes contribute to interior diapycnal diffusivity. They are
additive: each writes into the diffusivity arrays managed by
`MOM_set_diffusivity.F90`.

### Shear-driven mixing

`src/parameterizations/vertical/MOM_kappa_shear.F90` (Jackson, Hallberg,
Legg 2008) and the CVMix shear schemes
(`MOM_CVMix_shear.F90`) provide stratified-shear mixing.

```
USE_JACKSON_PARAM = True          # Jackson et al. 2008 kappa-shear
KAPPA_SHEAR_ITER_BUG = False
N_SMOOTH_PRES = 4
USE_CVMix_SHEAR = False           # Alternative: CVMix Pacanowski-Philander or KPP shear
```

### Internal-tide mixing

`src/parameterizations/vertical/MOM_tidal_mixing.F90` implements the
St Laurent 2002, Polzin 2009, and Melet 2012 internal-tide
parameterizations. Energy input fields are loaded by
`MOM_internal_tide_input.F90`. The propagating-internal-tide model lives
on the lateral side: `src/parameterizations/lateral/MOM_internal_tides.F90`.

```
INT_TIDE_DISSIPATION = True
INT_TIDE_PROFILE = "POLZIN_09"            # or "STLAURENT_02" or "MELET_2012"
TIDAL_DISSIPATION_FILE = "INPUT/tidal_dissipation.nc"
GAMMA_OSBORN = 0.2
```

### Double diffusion

`MOM_CVMix_ddiff.F90` provides salt-finger and diffusive-convective
double-diffusive mixing.

```
USE_CVMix_DDIFF = True
```

### Convection

Static convection: `MOM_CVMix_conv.F90` (CVMix style) and
`MOM_full_convection.F90` (legacy fallback). Only one is needed; the
diffusivity floor implied by full_convection guarantees neutral or
unstable columns are mixed.

### Background mixing

`MOM_bkgnd_mixing.F90` adds latitude- or N**2-dependent background
diffusivity profiles (Bryan-Lewis, Henyey-1 / -2, Decloedt-Luther).

```
KD = 1.5e-5                       # Background diffusivity (m^2/s)
KD_MIN = 2.0e-6
USE_LMD94 = True
HENYEY_IGW_BACKGROUND = False
```

### Bottom boundary layer

Bottom drag is set in `MOM_set_viscosity.F90`. Internal-wave drag from
rough topography is in `MOM_wave_drag.F90` (lateral subdir) and the
internal-tide energy budget.

### Geothermal heating

`MOM_geothermal.F90` adds a prescribed bottom geothermal heat flux:

```
GEOTHERMAL_SCALE = 1.0
GEOTHERMAL_FILE = "INPUT/geothermal_heating.nc"
```

### Sponges

`MOM_sponge.F90` (legacy, isopycnal) and `MOM_ALE_sponge.F90` (ALE-aware)
restore T, S, and tracers toward externally specified profiles. Used
heavily in regional configurations and in idealized cases that need
relaxation boundaries.

---

## Vertical: implicit friction and diffusion

`MOM_vert_friction.F90` solves the implicit vertical viscosity equation
for `u` and `v` at the end of every dynamical step. `MOM_diabatic_driver.F90`
solves the implicit vertical diffusion for `T`, `S`, and tracers. Both
use a tridiagonal solver (`MOM_kappa_shear.F90` and helpers).

---

## Vertical: surface forcing helpers

- `MOM_opacity.F90`: ocean-color-dependent shortwave penetration profile.
- `MOM_diabatic_aux.F90`: temperature- and salt-flux helpers, including
  river runoff routing into the surface layer.

---

## Lateral: viscosity

`src/parameterizations/lateral/MOM_hor_visc.F90` implements horizontal
viscosity in three flavors, optionally combined:

- **Laplacian** with constant or Smagorinsky / Leith coefficient.
- **Biharmonic** with constant, Smagorinsky, or Leith coefficient.
- **Negative Laplacian backscatter** driven by `MOM_MEKE`, which models
  the upscale energy cascade.

Switches:

```
LAPLACIAN = True
SMAGORINSKY_KH = True
SMAG_LAP_CONST = 0.06
BIHARMONIC = True
SMAGORINSKY_AH = True
SMAG_BI_CONST = 0.06
USE_LEITHY = False
```

A backscatter-aware setup adds `USE_KH_BG_2D = True` and `MEKE_BACKSCAT_RO = ...`
through `MOM_MEKE`.

---

## Lateral: thickness diffusion (Gent-McWilliams)

`MOM_thickness_diffuse.F90` implements Gent-McWilliams 1990 thickness
diffusion (the parameterized eddy-induced transport). Diffusivity
coefficients are assembled in `MOM_lateral_mixing_coeffs.F90` with
options including the Visbeck et al. 1997 scaling and a resolution
function that smoothly turns GM off as the grid becomes eddy-resolving.

```
THICKNESSDIFFUSE = True
THICKNESSDIFFUSE_FIRST = True       # Run before dynamics
KHTH = 200.0                        # GM thickness diffusivity (m^2/s)
USE_STORED_SLOPES = True
KHTH_USE_FGNV_STREAMFUNCTION = False
USE_VISBECK = True
VISBECK_L_SCALE = 50000.0
RESOLN_SCALED_KH = True
```

### Mesoscale Eddy Kinetic Energy (MEKE)

`MOM_MEKE.F90` and `MOM_MEKE_types.F90` implement the prognostic MEKE
sub-grid eddy energy budget (Jansen 2015, Marshall and Adcroft 2010,
Mak 2018). MEKE feeds `KHTH` (and biharmonic backscatter) so that
parameterized eddies respond to local conditions.

```
USE_MEKE = True
MEKE_DAMPING = 1.0e-7
MEKE_BACKSCAT_RO = 0.5
MEKE_GMCOEFF = 1.0
```

### Mixed-layer restratification

`MOM_mixed_layer_restrat.F90` implements two sub-mesoscale
restratification closures:

- **Fox-Kemper et al. 2008**: original mixed-layer eddy parameterization.
- **Bodner et al. 2023**: updated restratification scheme tied to the
  surface buoyancy flux.

```
MIXEDLAYER_RESTRAT = True
FOX_KEMPER_ML_RESTRAT_COEF = 0.06
USE_BODNER23 = False
```

### Generalized Mesoscale Eddy closure (GME, Zanna-Bolton)

`src/parameterizations/lateral/MOM_Zanna_Bolton.F90` implements the
Zanna-Bolton 2020 deterministic eddy-momentum-flux closure, including a
neural-network variant that uses `src/framework/MOM_ANN.F90`.

```
USE_ZB2020 = True
ZB_SCALING = 1.0
```

---

## Lateral: tides and self-attraction

- `MOM_tidal_forcing.F90`: astronomical tidal body forcing (M2, S2, K1,
  O1, ...). Tied to `TIDES = True`.
- `MOM_self_attr_load.F90` + `MOM_load_love_numbers.F90` +
  `MOM_spherical_harmonics.F90`: self-attraction and loading effect
  needed for accurate tidal solutions in high-resolution global runs.
- `MOM_internal_tides.F90`: prognostic internal-tide energy with
  topographic generation and dissipation.

---

## Lateral: filters

- `MOM_interface_filter.F90`: interface-thickness filter (layer mode).
- `MOM_streaming_filter.F90`: streaming filter for selected fields.

---

## Tracer-side parameterizations

In `src/tracer/`:

- `MOM_tracer_advect.F90` + `MOM_tracer_advect_schemes.F90`: PPM-based
  layered tracer advection.
- `MOM_neutral_diffusion.F90`: along-isopycnal (Redi) neutral diffusion.
- `MOM_hor_bnd_diffusion.F90`: horizontal boundary diffusion to handle
  diapycnal-mixing boundary effects.
- `MOM_tracer_hor_diff.F90`: top-level driver wiring above schemes into
  the tracer step.
- `MOM_tracer_diabatic.F90`: vertical (diabatic) tracer step (uses the
  same implicit solver as the temperature/salinity diabatic step).

---

## Sequencing inside the diabatic step

`MOM_diabatic_driver.F90` runs (in order):

```
diabatic
|-- set_diffusivity (assemble Kd from KPP, ePBL, kappa_shear, tidal mixing, bkgnd, ddiff)
|-- set_viscosity   (assemble Kv from KPP, ePBL, kappa_shear, drag)
|-- KPP_calculate / ePBL_column_main
|-- entrain_diffusive (isopycnal mode only)
|-- bulk_mixed_layer (isopycnal mode only)
|-- vertical advection of T, S, tracers (PPM)
|-- implicit vertical diffusion of T, S, tracers
|-- geothermal heating
|-- frazil melting
|-- ALE sponges
`-- vertvisc (implicit u, v friction)
```

The ordering matters: every parameterization that can write into Kd or
Kv must run before `set_diffusivity` returns.

---

## Adding a new vertical parameterization

1. Create `src/parameterizations/vertical/MOM_<name>.F90`. Define a
   `<name>_CS` (control struct) and the entry routines `<name>_init`,
   `<name>_calculate`, `<name>_end`.
2. Read parameters in `<name>_init` using `get_param(..., 'USE_<NAME>',
   ...)`. Log them so that they appear in `MOM_parameter_doc.all`.
3. Have your routine accumulate diffusivity into the existing arrays in
   `MOM_set_diffusivity.F90` (or `MOM_set_viscosity.F90`).
4. Register diagnostics through `MOM_diag_mediator` so the user can
   inspect `kd_<name>` etc.
5. Add a runtime guard in `MOM_diabatic_driver.F90`:
   ```fortran
   if (CS%use_<name>) call <name>_calculate(...)
   ```
6. Add a smoke test in `.testing/tc*` that flips `USE_<NAME> = True`.
7. Document in `docs/parameterizations_vertical.rst`.

The cleanest worked example is the Zanna-Bolton GME closure
(`MOM_Zanna_Bolton.F90`), which sits in the lateral subdir and includes a
neural-network branch that reuses `MOM_ANN.F90` from the framework.

---

## Where to next

- Tracer module overview: `architecture.md` (search `src/tracer/`)
- Vertical-coord engine: `vertical-coordinate.md`
- How to inspect what your parameterization did: `output-and-diagnostics.md`
- How to debug a parameterization that NaNs: `debugging.md`
