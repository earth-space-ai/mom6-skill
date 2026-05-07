# Running Experiments

Once you have a built MOM6 executable (`getting-started.md`), running an
experiment is a matter of preparing four text files (`MOM_input`,
`MOM_override`, `input.nml`, `diag_table`), an `INPUT/` directory of
forcing and grid files, and an empty `RESTART/` directory. This page
covers the layout the executable expects, the runtime-parameter system,
and how the bundled regression configurations under `.testing/` are
organized.

---

## Drivers and what each one expects

| Driver | Build location | Use case |
|--------|---------------|----------|
| `config_src/drivers/solo_driver/` | `ac/configure` ocean-only build | Standalone ocean. Built by `make` in your `build/` dir |
| `config_src/drivers/FMS_cap/` | GFDL FRE build system | GFDL CM4, OM4, SHiELD coupled |
| `config_src/drivers/nuopc_cap/` | CESM, UFS, E3SM build systems | NUOPC/ESMF coupled |
| `config_src/drivers/ice_solo_driver/` | Special build | Standalone ice-shelf |

The skill focuses on the `solo_driver` and the regression cases under
`.testing/` because they are the cleanest demonstrations of the
runtime-parameter system. See `coupling.md` for the coupled drivers.

---

## Experiment directory layout

A solo (ocean-only) experiment expects:

```
my_experiment/
|-- MOM6                       # symlink or copy of the built executable
|-- input.nml                  # FMS namelist (parameter_filename, diag_manager_nml, fms_nml)
|-- MOM_input                  # Baseline MOM6 parameters
|-- MOM_override               # Experiment-specific overrides (often empty)
|-- diag_table                 # FMS diagnostic table
|-- data_table                 # FMS data-override table (optional, often empty for solo)
|-- INPUT/
|   |-- ocean_hgrid.nc         # Horizontal supergrid
|   |-- ocean_topog.nc         # Bathymetry
|   |-- ocean_mask.nc          # Land mask
|   |-- vgrid.nc               # Vertical grid (interface positions)
|   |-- ocean_temp_salt_z.nc   # Initial T,S
|   |-- ocean.bcyc.YYYYMMDD.nc # Restart files (when restarting)
|   `-- ...                    # Forcing fields, tidal energy, geothermal, etc.
`-- RESTART/                   # Output restarts and diagnostic netCDF files appear here
```

For the canonical idealized configurations, see `.testing/tc1`..`tc4` in
the MOM6 source tree. For realistic configurations (regional, global),
see the `ocean_only/` and `ice_ocean_SIS2/` directories in
[MOM6-examples](https://github.com/NOAA-GFDL/MOM6-examples).

---

## `MOM_input` and `MOM_override`

`MOM_input` is the **baseline** parameter file. It contains every
non-default parameter for the experiment. `MOM_override` is read after
`MOM_input` and provides **deltas** from the baseline. The two-file split
makes it easy to compare experiments: stash the common configuration in
`MOM_input` and the experimental tweak in `MOM_override`.

Syntax (one parameter per line):

```
PARAMETER_NAME = value          ! optional units in [brackets] in trailing comment
ANOTHER = "string"
THIRD = 1.0e-3
LAYOUT = 4, 8                   ! Tuples for processor layouts
USE_THING = True                ! Booleans are True / False
```

`MOM_override` may use `#override` to silently replace a baseline value
with a different one:

```
#override DT = 1800.0
#override KHTH = 100.0
```

Without `#override`, repeating a parameter assignment with a different
value is a fatal error. With `#override`, it succeeds (and the change is
logged in `MOM_parameter_doc.all`).

### `MOM_parameter_doc.*`

Every run writes three files into the experiment directory:

| File | Contents |
|------|----------|
| `MOM_parameter_doc.all` | Every MOM6 parameter, with its description, units, default, and the value used |
| `MOM_parameter_doc.short` | Only parameters that differ from their default |
| `MOM_parameter_doc.layout` | Compile-time and decomposition parameters (NIPROC, NJPROC, ...) |

These are auto-generated. Treat them as the authoritative log of what
your run actually configured. Do not hand-edit them.

---

## `input.nml`

A minimal FMS namelist file looks like this (taken from `.testing/tc1`):

```fortran
&mom_input_nml
    output_directory = './'
    input_filename = 'n'
    restart_input_dir = 'INPUT/'
    restart_output_dir = 'RESTART/'
    parameter_filename =
        'MOM_input',
        'MOM_override',
/

&diag_manager_nml
/

&fms_nml
    clock_grain = 'ROUTINE'
    clock_flags = 'SYNC'
    domains_stack_size = 955296
    stack_size = 0
/
```

Important fields:

- `mom_input_nml%input_filename`: `'n'` for a fresh run, `'r'` for a
  restart from `INPUT/`.
- `mom_input_nml%parameter_filename`: ordered list of MOM6 parameter
  files to read. The order matters: later files override earlier ones.
- `&diag_manager_nml`: tunes the FMS diagnostic manager. See
  `output-and-diagnostics.md`.
- `&fms_nml`: domain-decomposition stack sizes. Increase `domains_stack_size`
  if FMS aborts with "stack exceeded".

---

## `diag_table`

Specifies the diagnostic output. Format (FMS diag_manager v2):

```
"experiment title"
1 1 1 0 0 0                      # base date YYYY MM DD HH MM SS

"prog",      1, "days", 1, "days", "time"
"prog_z",    1, "days", 1, "days", "time"
# files: name, freq, freq_units, file_format, time_units, calendar_type

"ocean_model",    "u",    "u",    "prog",   "all", .false., "none", 2
"ocean_model",    "v",    "v",    "prog",   "all", .false., "none", 2
"ocean_model",    "h",    "h",    "prog",   "all", .false., "none", 1
"ocean_model",    "temp", "temp", "prog",   "all", .false., "none", 2
"ocean_model_z",  "temp", "temp", "prog_z", "all", .false., "none", 2
# fields: module, field, output_name, file, time_avg, packing, ...
```

Note `ocean_model_z`: the `_z` suffix asks the diagnostic mediator to
remap the field to a fixed Z-grid before writing. `_rho` and `_sigma`
suffixes are also available. See `output-and-diagnostics.md`.

---

## `data_table`

Used by the FMS data-override system to feed forcing fields from netCDF
files into the model. Common in coupled and data-atmosphere modes; often
empty for the solo driver. Format (one entry per line):

```
"ATM", "t_bot", "t2m", "./INPUT/2t_ERA5.nc", "bilinear", 1.0
"OCN", "salt",  "",    "",                   "bilinear", 35.0   ! Constant override
```

Columns: `gridname`, `fieldname_code`, `fieldname_file`, `file_name`,
`interpol_method`, `factor`. A constant override is signaled by empty
`fieldname_file` and `file_name`; the `factor` then becomes the constant
value. YAML format is also supported.

---

## Running

Once the directory is laid out and the executable is in place:

```bash
cd my_experiment
mpirun -np 16 ./MOM6      # serial: ./MOM6
```

A clean run prints a banner with the build hash, then begins iterating.
Look for `STEP    1` in stdout. The model writes `ocean.stats` (global
energy and mass diagnostics every coupling step) and the netCDF files
declared in the `diag_table`.

Stop conditions are set in `MOM_input`:

```
TIMEUNIT = 86400.0                  ! [s] One model time unit = 1 day
DAYMAX = 5.0                        ! Stop after 5 model days
DT = 1800.0                         ! Baroclinic dt [s]
DT_THERM = 7200.0                   ! Thermodynamic dt [s] (must be a multiple of DT)
```

Restarts are written to `RESTART/` at the cadence set by
`RESTART_TIMESTAMPED = True` and `RESTINT`. To resume, copy the contents
of `RESTART/` into `INPUT/` and set `input_filename = 'r'` in
`input.nml`.

---

## The `.testing/tc*` reference cases

`.testing/` ships small, fast, idealized configurations that exercise
distinct parts of the code:

| Case | Purpose |
|------|---------|
| `tc0/` | Unit tests of model components |
| `tc1/` | Low-resolution `benchmark` configuration. ALE z* mode, GM, MEKE. The default smoke test |
| `tc1.a/` | tc1 with **un-split** Runge-Kutta 3 dynamics |
| `tc1.b/` | tc1 with un-split Runge-Kutta 2 dynamics |
| `tc2/` | ALE configuration with tides |
| `tc2.a/` | Sigma coordinate, PPM_H4, no tides |
| `tc3/` | Open-boundary-condition test (based on `circle_obcs`) |
| `tc4/` | Sponges and I/O initialization |

Each case has the same four-file layout (`MOM_input`, `MOM_override`,
`input.nml`, `diag_table`). They are excellent templates: copy `tc1/` to
a new directory, change the resolution and forcing, and you have a
starting point for a real experiment.

To run all of them and verify your build:

```bash
cd .testing
make -j               # builds .testing/build/symmetric/MOM6 etc.
make -j test          # runs grid, layout, restart, repro, dim, nan tests
```

See `.testing/README.rst` for the full configuration matrix and the
regression-testing knobs (`DO_REGRESSION_TESTS`, `DO_REPRO_TESTS`,
`DO_COVERAGE`).

---

## Real experiments via MOM6-examples

Realistic experiments live in
[MOM6-examples](https://github.com/NOAA-GFDL/MOM6-examples). Highlights:

| Path | What it is |
|------|-----------|
| `ocean_only/double_gyre/` | Idealized double-gyre. Good first realistic case |
| `ocean_only/benchmark/` | The benchmark `.testing/tc1` is derived from |
| `ocean_only/global_ALE/OM4_025/` | OM4 quarter-degree global, z* coordinate |
| `ocean_only/global_ALE/baseline/` | OM4 with a reduced parameter set |
| `ocean_only/Neverworld2/` | The NeverWorld2 idealized basin |
| `ice_ocean_SIS2/OM4_025/` | OM4_025 coupled to SIS2 sea ice |
| `ice_ocean_SIS2/Baltic_OM5/` | Regional Baltic test case |
| `coupled_AM2_LM3_SIS2/` | Fully coupled atmos-land-ocean-ice |

Each directory ships a `MOM_input`, `MOM_override`, `input.nml`,
`diag_table`, and a build script template. The
[MOM6-examples wiki](https://github.com/NOAA-GFDL/MOM6-examples/wiki)
documents per-platform build instructions, especially for NCAR Derecho,
GFDL Gaea, and NOAA Hera.

---

## Where to next

- Diagnostics in detail: `output-and-diagnostics.md`
- Add a parameter or change a scheme: `physics-parameterizations.md`
- Coupled (CESM/E3SM/CM4) workflow: `coupling.md`
- Crash on first time step: `debugging.md`
