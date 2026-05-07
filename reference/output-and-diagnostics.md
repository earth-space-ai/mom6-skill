# Output and Diagnostics

MOM6 emits diagnostic output through GFDL's FMS `diag_manager` library.
The user controls what is written via the `diag_table` text file. MOM6
adds a layer on top: the **diagnostic mediator**
(`src/framework/MOM_diag_mediator.F90`) that lets a single registered
field be written in native (model-layer) coordinates and in remapped
Z-, rho-, or sigma-space output streams simultaneously, with no per-field
code changes.

This page covers the `diag_table` grammar, the native-vs-remapped
machinery, the `ocean.stats` and `chksum_diag` text outputs, and restart
files.

---

## How diagnostic output is plumbed

```
parameterization or core module
      |
      v
register_diag_field("ocean_model", "temp", axes_h, ...)
      |
      v
post_data(handle, field_array, diag_CS)
      |
      v
MOM_diag_mediator         <-- decides: native, _z, _rho, _sigma copy
      |
      v
FMS diag_manager          <-- writes netCDF in line with diag_table
```

The key call is `post_data(diag_id, field, diag_CS)`. The mediator looks
at every registered output stream that requested this diagnostic. If the
stream has a `_z` (or `_rho`, `_sigma`) suffix on its module name, the
mediator passes the field through `MOM_diag_remap` first.

---

## `diag_table` grammar (FMS diag_manager v2)

A typical `diag_table` has three sections. Lines starting with `#` are
comments.

```
"experiment_label"            ! Free-text label written into every output file
1900 1 1 0 0 0                ! Reference (base) date YYYY MM DD HH MM SS

# Files: name, freq, freq_units, file_format, time_units, calendar_type
"prog",       1, "days", 1, "days", "time"
"prog_z",     1, "days", 1, "days", "time"
"forcing",    1, "days", 1, "days", "time"

# Fields: module, field, output_name, file, time_avg, packing, ...
"ocean_model",    "u",     "u",     "prog",     "all", .true.,  "none", 2
"ocean_model",    "v",     "v",     "prog",     "all", .true.,  "none", 2
"ocean_model",    "h",     "h",     "prog",     "all", .true.,  "none", 1
"ocean_model",    "temp",  "temp",  "prog",     "all", .true.,  "none", 2
"ocean_model",    "salt",  "salt",  "prog",     "all", .true.,  "none", 2

# Z-remapped diagnostics (note module suffix _z)
"ocean_model_z",  "temp",  "temp",  "prog_z",   "all", .true.,  "none", 2
"ocean_model_z",  "salt",  "salt",  "prog_z",   "all", .true.,  "none", 2

# Rho-remapped (sigma2 surfaces)
"ocean_model_rho2", "u",   "u",     "prog_rho", "all", .true.,  "none", 2
```

Field-line columns:

| Column | Meaning |
|--------|---------|
| `module` | Diagnostic module string. Native: `"ocean_model"`. Remapped: `"ocean_model_z"`, `"ocean_model_rho2"`, `"ocean_model_sigma"`, etc. |
| `field` | Internal name registered via `register_diag_field` |
| `output_name` | Name written to the netCDF variable |
| `file` | Which file (from the file section) to write to |
| `time_sampling` | `"all"` for every sample, `"none"` for snapshots |
| `time_avg` | `.true.` for time-mean, `.false.` for snapshots |
| `other_ops` | Usually `"none"` |
| `packing` | netCDF packing: `1` = single-precision, `2` = double-precision, `4` = 16-bit packed |

Aliases like `"ocean_model_z_new"` exist for stepwise variants. See
`MOM_diag_mediator.F90` for the full list of supported suffixes.

---

## Native vs remapped diagnostics

MOM6 stores prognostic state on its native (Lagrangian) layers, where
layer thickness `h` varies with time and the vertical coordinate. For
analysis, you almost always want fields on a fixed coordinate.

The `_<COORD>` suffix on the module name triggers diagnostic remapping:

| Suffix | Coordinate | Module |
|--------|-----------|--------|
| `_z` | Fixed depth (m) | `MOM_diag_remap` configured by `DIAG_REMAP_Z_GRID_DEF` |
| `_rho2` | Potential density referenced to 2000 dbar | `DIAG_REMAP_RHO2_GRID_DEF` |
| `_sigma` | Terrain-following | `DIAG_REMAP_SIGMA_GRID_DEF` |

The target grid for each remap stream is configured in `MOM_input`:

```
NUM_DIAG_COORDS = 3
DIAG_COORDS = "z 01 ZSTAR", "rho2 02 RHO", "sigma 03 SIGMA"
DIAG_COORD_DEF_01 = "FILE:vgrid_z.nc,interfaces=zw"
DIAG_COORD_DEF_02 = "FILE:rho2_targets.nc,interfaces=rho2"
DIAG_COORD_DEF_03 = "UNIFORM:75"
```

`DIAG_COORD_DEF_NN` accepts the same coordinate-config strings as
`ALE_COORDINATE_CONFIG`: `FILE:`, `UNIFORM:`, `HYBRID:`, etc.

When a diagnostic is requested as `"ocean_model_z"` for `temp`, the
mediator vertically integrates the native-layer profile against the Z
grid using the same conservative remapping kernel as the ALE step. The
result is a true layer-integrated mean on the Z bins, not an
interpolation.

---

## Where diagnostics are registered

Search for `register_diag_field` to find where each variable is exposed.
A typical pattern:

```fortran
! In MOM_PressureForce_FV.F90
CS%id_temp = register_diag_field('ocean_model', 'temp', diag%axesTL, Time, &
     'Potential Temperature', 'degC')

! Later, inside the routine:
if (CS%id_temp > 0) call post_data(CS%id_temp, temp, CS%diag)
```

If you add a new diagnostic, follow this exact pattern: store the handle
in your CS, register in your `_init`, post in your routine. The
`post_data` call is a no-op when the handle is `<= 0` (i.e. the user did
not request it in `diag_table`).

---

## Text-mode diagnostics

In addition to netCDF output, MOM6 emits two text files at runtime:

### `ocean.stats`

Written by `src/diagnostics/MOM_sum_output.F90`. One line per coupling
step (or every `ENERGYSAVEDAYS` model time units in the solo driver):

```
Step,  Day,  Truncations, Energy/Mass [m2 s-2],   Maximum CFL,  Mean Sea Level [m],  Total Mass [kg], Mean Salin [PSU], Mean Temp [C], Frac Mass Lost
   0,    0.0000,         0,  0.000000000000000E+00,  0.00000,  0.0000000000000E+00,  1.36e+21, 34.7299, 3.5950, 0.000e+00
   1,    0.0104,         0,  3.142857142857142E-04,  0.04127, ...
```

The total energy column is reported at machine precision; everything
else is at lower precision. `ocean.stats` is the primary check on
numerical health: drift in mean temperature or total mass means a
budget is broken.

`MOM_sum_output` writes a more verbose text summary into `MOM6.stdout`
on coupled runs.

### `chksum_diag`

Bit-identical checksums of state arrays at coupling steps. Used by the
`.testing/` regression suite to detect bit changes between transformed
and reference runs. Set `DEBUG_CHKSUMS = True` for more frequent dumps.

### Truncation diagnostic files

If `MAXTRUNC > 0`, `MOM_PointAccel.F90` writes per-cell dumps of the
momentum equation when |u| or |v| exceeds the truncation threshold.
Files: `U_truncations.<PE>` and `V_truncations.<PE>`. Each entry shows
every term in the momentum budget at that grid point at the moment of
truncation. This is the single most useful crash-debugging output. See
`debugging.md`.

---

## Restart files

`MOM_restart.F90` coordinates writing and reading restart files to
`RESTART/` (output) and reading from `INPUT/` (input). Two key
parameters:

```
RESTART_TIMESTAMPED = True               ! Append YYYYMMDD to restart names
RESTINT = 30.0                           ! Write a restart every RESTINT TIMEUNITs
ROBUST_RESTART_READ = True
```

A typical restart name is `MOM.res.YYYYMMDD-HHMMSS.nc` plus per-tracer
files. A run that completes will also write a final restart whose
timestamp matches the run's last step.

Bit-for-bit restart is part of the regression suite (`.testing/test.restart`):
a single 12-hour run must produce identical state to two 6-hour runs
separated by a restart. Failures here indicate that some module is not
correctly registering its prognostic state with `register_restart_field`.

---

## I/O performance

- `IO_LAYOUT = X, Y` in `MOM_input` sets the I/O processor layout for
  parallel netCDF. `0, 0` defaults to the same layout as the compute
  decomposition.
- Use the FMS2 infra wrapper (`config_src/infra/FMS2/`) for the modern
  PIO-based I/O. It is significantly faster for high-resolution
  configurations than FMS1.
- Set `PARALLEL_RESTARTFILES = True` to write per-PE restarts and combine
  them in post-processing. Avoids the gather to root.

---

## Debugging diagnostics

Several knobs in `MOM_input` raise the verbosity:

```
DEBUG = True                       ! Enable debugging code paths globally
DEBUG_TRUNCATIONS = True           ! Print every truncation, not just count
CHECK_BAD_SURFACE_VALS = True      ! Sanity-check surface T/S each step
DEBUG_CHKSUMS = True               ! More frequent checksum dumps
DEBUG_REDUNDANT = True             ! Validate halo updates
```

These slow the model down significantly; turn off for production.

---

## Where to next

- Set up an experiment that uses these outputs: `running-experiments.md`
- Coordinate background for the `_z` / `_rho` remapping: `vertical-coordinate.md`
- Common signs your output is broken: `debugging.md`
- Add a new diagnostic field: `architecture.md` (search `register_diag_field`)
