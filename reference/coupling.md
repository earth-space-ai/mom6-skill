# Coupling

MOM6 ships in many large coupled Earth-system models. The single MOM6
source tree supports four driver layers, each in its own subdirectory of
`config_src/drivers/`:

| Driver | Coupler | Used by |
|--------|---------|---------|
| `solo_driver/` | None (ocean-only) | Standalone runs, `.testing/`, idealized work |
| `FMS_cap/` | GFDL FMS coupler | NOAA-GFDL CM4, OM4, SHiELD, SPEAR |
| `nuopc_cap/` | NUOPC + ESMF | NCAR CESM2/CESM3, NOAA UFS, DOE E3SM |
| `STALE_mct_cap/` | MCT | Legacy CESM (deprecated) |

All four sit on top of an unchanged `src/`. The driver layer translates
the host coupler's data structures into the `forcing` and `surface`
derived types defined in `src/core/MOM_forcing_type.F90` and
`src/core/MOM_variables.F90`, calls `step_MOM`, and translates the
results back.

This page maps each driver and points at the production systems that
consume them.

---

## `solo_driver/` (ocean-only)

Files in `config_src/drivers/solo_driver/`:

```
MOM_driver.F90              ! Main program
MOM_surface_forcing.F90     ! Builds the forcing struct from data files
user_surface_forcing.F90    ! Per-experiment overrides
MESO_surface_forcing.F90    ! MESO benchmark forcing
atmos_ocean_fluxes.F90      ! Bulk-formula helpers
```

The main loop:

```fortran
do while (Time < Time_end)
   call set_forcing(...)            ! Read or compute surface forcing
   call step_MOM(forces, fluxes, sfc_state, Time, dt, ...)
   call write_diag_data(...)
   call advance_time(Time, dt)
enddo
```

Built by `ac/configure` + `make`. This is what you get from the autoconf
quick start.

---

## `FMS_cap/` (GFDL coupled)

Files in `config_src/drivers/FMS_cap/`:

```
ocean_model_MOM.F90              ! Implements the FMS ocean_model_type API
MOM_surface_forcing_gfdl.F90     ! Surface flux assembly from FMS coupler
```

The FMS cap implements the public API that the GFDL coupler expects
(`ocean_model_init`, `update_ocean_model`, `ocean_model_end`, ...). Used
by:

- **CM4**: GFDL Coupled Model 4 (CMIP6 production model)
- **OM4**: ocean-ice spin-up version of CM4
- **SHiELD**: NOAA System for High-resolution prediction on Earth-to-Local Domains
- **SPEAR**: GFDL seasonal-to-decadal prediction system

Builds with the FRE workflow (`fremake`, `frerun`) at GFDL or with the
manual `mkmf` flow elsewhere. See the
[MOM6-examples wiki](https://github.com/NOAA-GFDL/MOM6-examples/wiki).

---

## `nuopc_cap/` (NUOPC/ESMF coupled)

Files in `config_src/drivers/nuopc_cap/`:

```
ocn_comp_NUOPC.F90               ! NUOPC component (registers Init/Run/Finalize)
mom_cap.F90                      ! Cap module bridging to MOM6 internals
mom_cap_methods.F90              ! Field exports/imports, advertise/realize
mom_cap_outputlog.F90            ! Output-log helpers
mom_cap_profiling.F90            ! ESMF profiling integration
mom_cap_time.F90                 ! ESMF clock <-> FMS time conversion
mom_inline_mod.F90               ! Inline run helpers
mom_ocean_model_nuopc.F90        ! NUOPC-aware ocean_model_type
mom_surface_forcing_nuopc.F90    ! Surface flux assembly from NUOPC mediator
time_utils.F90                   ! Time bookkeeping
```

The NUOPC cap implements the standard ESMF NUOPC component contract.
Field exports include `So_t` (sea surface temperature), `So_s` (sea
surface salinity), `So_u`, `So_v` (surface currents), `So_dhdx`, `So_dhdy`
(sea-surface-height tilt for atmosphere). Imports include momentum,
heat, freshwater, and shortwave fluxes from the mediator.

Used by:

- **NCAR CESM2 / CESM3**: through CESM's `ocn` component, replacing POP
- **NOAA UFS** (Unified Forecast System): MOM6 is the ocean of the
  coupled subseasonal-to-seasonal applications
- **DOE E3SM**: MOM6 is being adopted as a future ocean component

The CESM-MOM6 integration is the most complete: see the CESM source
tree under `components/mom/` (CESM3) for the connecting layer.

---

## `STALE_mct_cap/` (deprecated)

Legacy MCT cap used by older CESM versions. The directory is prefixed
`STALE_` to flag that it is unsupported. Newer CESM versions (CESM2 and
later) use NUOPC.

---

## Surface-forcing types

All four drivers populate the same shared types defined in
`src/core/MOM_forcing_type.F90`:

```fortran
type, public :: forcing
  real, allocatable :: ustar(:,:)                ! Friction velocity
  real, allocatable :: tau_x(:,:), tau_y(:,:)    ! Wind stresses
  real, allocatable :: lprec(:,:), fprec(:,:)    ! Liquid + frozen precip
  real, allocatable :: evap(:,:), lrunoff(:,:), frunoff(:,:)
  real, allocatable :: sw(:,:), lw(:,:)          ! Net SW, net LW
  real, allocatable :: sens(:,:), latent(:,:)    ! Sensible + latent heat
  real, allocatable :: salt_flux(:,:)            ! Sea-ice salt rejection
  type(coupler_2d_bc_type) :: tr_fluxes          ! Tracer surface fluxes
  ...
end type
```

The driver-specific `MOM_surface_forcing*.F90` modules allocate and fill
this struct. The core (`step_MOM`) consumes it without caring which
coupler produced it.

---

## Data tables and `data_override`

When a driver wants to feed forcing from disk (data atmospheres,
constant fluxes, observed SST), it routes through FMS's
`data_override` machinery, controlled by `data_table`. Format:

```
"ATM", "t_bot", "t2m", "./INPUT/2t_ERA5.nc", "bilinear", 1.0
"ATM", "u_bot", "u10", "./INPUT/10u_ERA5.nc", "bilinear", 1.0
"OCN", "salt",  "",    "",                    "bilinear", 35.0   ! Constant
```

Columns: `gridname`, `fieldname_code`, `fieldname_file`, `file_name`,
`interpol_method`, `factor`. Empty `fieldname_file` and `file_name`
trigger constant-value override (the `factor` becomes the value). The
data atmosphere of an ocean-ice run (e.g. JRA55-do or ERA5) is
specified entirely through `data_table` entries.

YAML is also supported. See the FMS data_override
[documentation](https://github.com/NOAA-GFDL/FMS/tree/main/data_override).

---

## Cross-component flux conservation

Heat and freshwater conservation across the coupling boundary is the
single most common source of long-term drift in coupled MOM6 runs.
Diagnostic checks:

- `ocean.stats` reports `Frac Mass Lost`: should be O(1e-12) per step.
- `ocean.stats` reports total heat content drift.
- The coupler's own log (FMS `ocean_solo`, NUOPC mediator, CESM `cpl.log`)
  should show component flux totals that balance.

`MOM_diagnose_KdWork.F90` and `MOM_sum_output.F90` are the budget
diagnostic modules to consult when conservation breaks.

---

## Adding a new coupler driver

The pattern (used by both FMS_cap and nuopc_cap):

1. Create a new directory under `config_src/drivers/<your_driver>/`.
2. Write a top-level module that implements your coupler's component
   contract (init, advance, finalize).
3. Wrap a `forcing` and `surface` instance around the host coupler's
   field representations.
4. Call into `MOM.F90`'s public API: `initialize_MOM`, `step_MOM`,
   `MOM_state_save_for_restart`, `MOM_end`.
5. Add a build-rule fragment for your coupler's build system.

The `config_src/external/` null modules let you compile driver code that
references optional packages (BGC, ODA, MARBL) without their real
implementations being present.

---

## ESMF caps and ESM frameworks

NUOPC (National Unified Operational Prediction Capability) is built on
ESMF (Earth System Modeling Framework). The MOM6 NUOPC cap is the
primary entry point for ESMF-based coupled systems. It registers field
"advertise" calls during init, then "realizes" each field once the
mediator declares the connections. The cap supports:

- ESMF clocks for synchronization with the mediator
- ESMF profiling and tracing
- ESMF logs (separate from FMS logs)

The cap is regularly updated to track upstream NUOPC changes; check the
`config_src/drivers/nuopc_cap/` history before pinning a MOM6 commit
into a new ESM build.

---

## Where to next

- Build the ocean-only solo driver: `getting-started.md`
- The `forcing` and `surface` types: `architecture.md`
- Conservation diagnostics: `output-and-diagnostics.md` and `debugging.md`
- Submit changes to the cap: `contributing-pr.md`
