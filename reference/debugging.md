# Debugging MOM6

This page covers the runtime failure modes you will see most often in
MOM6: build link errors, NaN floods, CFL truncations, broken salt or
heat budgets, and bit-for-bit restart mismatches. Each section lists the
diagnostic to check first and the parameter knobs that raise verbosity.

---

## Tier 1: build-time errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| `pkg/CVMix-src/...: No such file` | Submodules not initialized | `git submodule update --init --recursive` |
| `cannot find -lnetcdff` | Fortran netCDF bindings missing | Install `netcdf-fortran` separately from `netcdf-c` |
| `mpif90: command not found` | MPI not in `PATH` | `module load openmpi` (or set `MPIFC=...`) |
| `Could not find FMS module file fms.mod` | FMS not built | `make -C ac/deps`, or set `FCFLAGS=-I/path/to/fms/include` |
| `undefined reference to mpp_*` | FMS link missing | Add `LDFLAGS=-L/path/to/fms/lib -lFMS` |
| `Error: Symbol "mom6_register_diag_field" at (1) is the name of a procedure` | Module name clash from FMS1/FMS2 mix | Pick exactly one of `config_src/infra/FMS1` or `FMS2` |

---

## Tier 2: runtime crashes

### "STOP: NaNs in ..."

The model wraps fatal NaN errors through `MOM_error_handler`. The
message will name the offending module and array. Common sources:

- **Unstable initial state.** Set `DEBUG = True` in `MOM_input` to
  re-run with extra checking. Verify your IC files have realistic T, S,
  and that the bathymetry has no unmasked land cells.
- **CFL violation.** The Lagrangian dynamical step can blow up before
  the truncation logic catches it. Drop `DT` by half and rerun.
- **Bad surface forcing.** Set `CHECK_BAD_SURFACE_VALS = True` to abort
  with an informative message when forcing fields are out of range.

Build with NaN-trapping flags so the abort happens at the first
floating-point exception:

| Compiler | Add to `FCFLAGS` |
|----------|------------------|
| gfortran | `-g -O0 -fbacktrace -ffpe-trap=invalid,zero,overflow -fcheck=bounds -finit-real=snan` |
| Intel ifort/ifx | `-g -O0 -traceback -fpe0 -check all -init=snan` |

These slow the model 2-5x. Use only for debugging.

### "U_TRUNC: too large velocity at ..."

A grid cell exceeded the velocity ceiling
`MAXVEL`. Defaults are around 10 m/s. The truncation is recorded in
`U_truncations.<PE>` and `V_truncations.<PE>` files written by
`src/diagnostics/MOM_PointAccel.F90`. These files dump every term in
the momentum budget for the offending column at the offending step:

```
============= U-velocity truncation at i=43 j=87 k=12 =============
  U  before:   1.234e-01    U after:   1.001e+01
  CAu (Coriolis-advection): -2.34e-04
  PFu (pressure force):      4.56e-03
  Diffu (horizontal visc):  -1.23e-05
  visc_rem (vert visc):      9.99e-01
  ...
```

Look at which term is large. Common patterns:

- Pressure force dominates -> bathymetry has a single-cell feature
  generating a spurious pressure gradient. Smooth or remove.
- Coriolis-advection dominates -> upstream wind stress spike. Filter.
- Vertical viscosity remnant near 1.0 -> implicit solver did nothing,
  meaning Kv is near zero where it should not be. Check KPP/ePBL setup.

`MAXTRUNC` in `MOM_input` controls how many truncations are tolerated
before fatal abort. Default is small (10s). Setting it large lets the
run continue and produce many `U_truncations` files for analysis.

### Segfault in `MOM_dynamics_split_RK2.F90`

Almost always an array-bounds issue caught by `-fcheck=bounds` (gfortran)
or `-check all` (Intel). Rebuild with debug flags. The traceback will
name the offending line. Common causes:

- Halo not updated before a stencil read. Look for missing `pass_var`.
- Open-boundary array touched outside its allocated range.
- New diagnostic field registered with the wrong axis triple.

---

## Tier 3: silent numerical drift

### `ocean.stats` shows total mass changing

Open `ocean.stats` after a run that should have been mass-conservative.
The `Frac Mass Lost` column should be O(1e-12) per step. Larger numbers
mean a flux is being silently double-counted or dropped:

- Mismatch between `forcing%lprec`, `evap`, `runoff`, and the salinity
  flux.
- A new tracer added without its surface flux being declared.
- Coupler-side mismatch between freshwater and salt flux conventions.

Set `CALCULATE_DIAGNOSTIC_TENDENCIES = True` to get tendency
diagnostics for every term in the T/S budget. Cross-check the sum
against the actual time derivative.

### `ocean.stats` shows total heat content drifting

Same pattern. Suspect:

- `frazil` heat flux not entering the budget correctly.
- Geothermal heat flux scale mis-set (`GEOTHERMAL_SCALE`).
- River runoff temperature not zero where it should be (or vice versa).
- Shortwave penetration profile inconsistent with the `MOM_opacity`
  configuration.

The aggregate energy diagnostic (`MOM_diagnose_KdWork.F90`) breaks down
the energy of the column into BPE, APE, KE, and pieces of the diabatic
contribution. Use it to localize which budget term is leaking.

---

## Tier 4: bit-for-bit restart failures

The `.testing/test.restart` test runs a 12-hour reference simulation
and compares it to two 6-hour halves separated by a restart. They must
match to the bit. Failure means some prognostic state is not being
checkpointed.

Diagnostic procedure:

1. Re-run with `DEBUG_CHKSUMS = True`. Compare the per-coupling-step
   checksums in `chksum_diag` between the reference and restart paths.
   The first divergent timestep tells you when the missing field
   matters.
2. The first divergent **field** (look at the chksum field name) tells
   you which array was wrong. Common sources:
   - A new prognostic state added to a `*_CS` (control struct) without
     a corresponding `register_restart_field` call in the module's
     `_init` routine.
   - A diagnostic accumulator that reset incorrectly across restart.
   - Random-number state not restarted (use `MOM_random.F90` interfaces).
3. Add the missing `register_restart_field(restart_CS, "myname",
   array, longname=..., units=...)`. Rerun.

`PARALLEL_RESTARTFILES = True` writes per-PE restarts (faster) but
sometimes hides PE-decomposition-related restart issues. Test with
single-file restarts at least once per change.

---

## Tier 5: regression-test failures

Run `make -j test` in `.testing/` with a recent `dev/gfdl` checked out:

```
PASS  tc1.symmetric.grid
PASS  tc1.symmetric.layout
FAIL  tc1.symmetric.restart
PASS  tc1.symmetric.repro
...
```

| Failed test | What to look at |
|-------------|-----------------|
| `*.grid` | Symmetric vs nonsymmetric grids differ. Check halo updates and OBC code |
| `*.layout` | Serial vs parallel differ. Look for non-reproducing reductions or missing halo updates |
| `*.restart` | Restart not bit-identical. See Tier 4 |
| `*.repro` | DEBUG vs REPRO build differ. An optimization is breaking a reproducibility property. Lower `FCFLAGS_REPRO` to `-O2` and check parenthesization |
| `*.dim` | Dimensional rescaling differs. A new module is missing dimensional scaling factors. See `MOM_unit_scaling.F90` |
| `*.nan` | NaN-init test caught uninitialized memory. Initialize the offending array in your module's `_init` |

For dimensional rescaling: every real argument in MOM6 carries a
dimensional scale factor (`US%T_to_s`, `US%L_to_m`, `US%R_to_kg_m3`, ...).
The `.dim` test multiplies and divides every scale by 2 in turn. If
your module hardcodes a unit conversion or drops a scale factor, the
output diverges.

---

## Useful debugging knobs in `MOM_input`

```
DEBUG = True                         ! Master debug switch
DEBUG_TRUNCATIONS = True             ! Print every truncation, not just count
DEBUG_CHKSUMS = True                 ! More frequent checksum output
DEBUG_REDUNDANT = True               ! Validate halo updates after each pass
CHECK_BAD_SURFACE_VALS = True        ! Abort on out-of-range surface forcing
BAD_VAL_SST_MIN = -2.0
BAD_VAL_SST_MAX = 50.0
BAD_VAL_SSS_MIN = 0.0
BAD_VAL_SSS_MAX = 50.0
WRITE_GEOM = 1                       ! Dump geometry to ocean_geometry.nc
```

For routine performance debugging:

```
DT = 900.0                           ! Halve to test if truncations are dt-driven
DT_THERM = 7200.0                    ! Set to a multiple of DT
DTBT = -0.95                         ! Negative = auto-set as fraction of dt
```

---

## When all else fails

1. Reproduce the failure on a tiny configuration. Strip your run to the
   smallest grid and shortest interval that triggers the bug.
2. Bisect against `dev/gfdl`. Find the commit that introduced the
   regression with `git bisect run`.
3. Open a discussion on
   [NOAA-GFDL/MOM6 Discussions](https://github.com/NOAA-GFDL/MOM6/discussions)
   with the truncation file or `chksum_diag` excerpt and the smallest
   reproducing inputs.

---

## Where to next

- Build flags reference: `getting-started.md`
- The diagnostic mediator that powers `chksum_diag`: `output-and-diagnostics.md`
- The dynamical core call chain: `architecture.md`
- Submit your fix: `contributing-pr.md`
