# MOM6 Skill

A progressive-disclosure skill for [MOM6](https://github.com/NOAA-GFDL/MOM6),
the Modular Ocean Model version 6, developed by NOAA-GFDL and a multi-institution
consortium.

> **Maintainer of MOM6:** NOAA-GFDL plus consortium (Princeton, NCAR, NOAA-EMC, DOE-E3SM)
> **Skill author:** Koutian Wu (ktwu01@gmail.com)
> **Skill version:** 0.1.0

## What This Is

A self-contained knowledge package that teaches AI agents (and humans) how to
**build, run, modify, debug, couple, and contribute to** MOM6.

The skill captures the procedural knowledge that is normally only transmitted
by working alongside an experienced MOM6 developer: how the ALE engine
separates the dynamics from the vertical grid, why `MOM_input` and
`MOM_override` are split, how to ask the diag_manager for Z-space output, why
ocean-only and coupled builds use different drivers, and how to keep your
patch within the reproducibility rules of the dynamical core.

**Progressive disclosure:**
- `SKILL.md`: routing hub. Decision tree, repo layout, critical rules, quick start.
- `reference/*.md`: deep-dive docs loaded on demand.

## Contents

| Document | What's inside |
|----------|---------------|
| `SKILL.md` | Entry point. Decision tree, repo layout, quick start, critical rules |
| `reference/getting-started.md` | Two-repo layout, FMS dependency, autoconf path, .testing smoke test |
| `reference/architecture.md` | `src/` and `config_src/` walkthrough, dynamical-core call chain, key derived types |
| `reference/vertical-coordinate.md` | ALE engine: regrid, remap, coord_*, reconstructions, hybrid HYCOM |
| `reference/physics-parameterizations.md` | KPP, ePBL, kappa-shear, internal-tide mixing, GM, MEKE, neutral diffusion, GME |
| `reference/running-experiments.md` | MOM_input + MOM_override, diag_table, data_table, .testing layouts |
| `reference/output-and-diagnostics.md` | FMS diag_manager, native vs Z/rho/sigma remapped output, ocean.stats |
| `reference/coupling.md` | solo, FMS_cap, NUOPC, MCT (deprecated). MOM6 inside CESM, E3SM, UFS, SHiELD, CM4 |
| `reference/debugging.md` | NaNs, CFL, salt/heat budgets, MOM_PointAccel, restart bit-for-bit failures |
| `reference/contributing-pr.md` | Fork, dev/gfdl, .testing regression suite, code style, Doxygen |

## Sources

This skill is grounded in:

1. The **NOAA-GFDL/MOM6** repository (main branch) source tree, including
   `README.md`, `ac/README.md`, `.testing/README.rst`, and the Sphinx docs
   under `docs/`.
2. **mom6.readthedocs.io**: the public Sphinx documentation site.
3. The **NOAA-GFDL/MOM6/wiki** pages on code style, developer workflow,
   diagnostic vertical remapping, runtime parameter system, and Doxygen.
4. **Adcroft et al. 2019**, "The GFDL Global Ocean and Sea Ice Model OM4.0:
   Model Description and Simulation Features", JAMES 11:3167-3211,
   doi:10.1029/2019MS001726, the canonical MOM6 description paper.
5. The **NOAA-GFDL/MOM6-examples** repo and its wiki for experiment layouts.

## Install

This skill follows the same layout as
[laps-skill](https://github.com/huangzesen/laps-skill), the xhelio family
(`xhelio-cdaweb`, `xhelio-spice`, `xhelio-pds`), and the noahmp-skill:

```
mom6-skill/
|-- SKILL.md               # routing hub (read first)
|-- README.md              # this file
|-- LICENSE
`-- reference/             # deep-dive docs
```

To use with a Claude Code or LingTai agent, drop the directory into your
skills library and refresh.

## License

MIT. MOM6 itself is governed by the Apache License 2.0; see
https://github.com/NOAA-GFDL/MOM6/blob/main/LICENSE.md.
