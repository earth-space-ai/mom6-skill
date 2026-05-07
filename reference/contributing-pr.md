# Contributing a Pull Request

MOM6 development is a fork-and-pull-request workflow on GitHub against
`NOAA-GFDL/MOM6`. The development branch is `dev/gfdl`; pull requests
target it. The CI suite is layered: GitHub Actions runs the basic build
and the `.testing/` regression suite, while a mirrored GFDL GitLab
pipeline runs the heavier-weight platform tests. This page walks the
process from fork to merge and lists the rules that catch most
first-time contributors.

---

## Fork model

```
NOAA-GFDL/MOM6  (upstream, dev/gfdl is the active branch)
       ^
       | upstream
       |
   <yourname>/MOM6   (your fork)
       ^
       | origin
       |
   local clone with --recurse-submodules
```

```bash
git clone --recurse-submodules https://github.com/<yourname>/MOM6
cd MOM6
git remote add upstream https://github.com/NOAA-GFDL/MOM6
git fetch upstream
git checkout -b feature/my-change upstream/dev/gfdl
```

Always branch off `upstream/dev/gfdl`, never `main`. `main` lags behind
`dev/gfdl` and is updated periodically when GFDL cuts a release.

---

## Make your change

### Code style

Follow the
[MOM6 code style guide](https://github.com/NOAA-GFDL/MOM6/wiki/Code-style-guide).
The rules that bite hardest:

- 2-space indentation. No tabs. No trailing whitespace.
- 100-character line target, 120-character hard limit.
- `implicit none ; private` at the top of every module.
- All public types and routines documented with Doxygen `!>` and `!<`.
- `intent(in)`, `intent(out)`, or `intent(inout)` on every dummy argument.
- All real variables get a unit comment.
- snake_case names. Multi-word, descriptive (`del_rho_int`, not `drho`).
- No global module data (rare debug exceptions).

### Reproducibility discipline

Sums must be MPI-reproducible. The dynamical core forbids:

- Bare `sum()`, `prod()`, `matmul()` over arrays.
- Unparenthesized chained additions: `a + b + c + d` is wrong; write
  `(a + b) + (c + d)`.
- Transcendental functions (`exp`, `log`) where they could be avoided.
- The `**` exponent operator with non-integer powers in hot loops.

For partial-sum reductions across PEs, use `reproducing_sum` from
`MOM_coms.F90`, not `mpp_sum`. The `.testing/test.layout` and
`test.repro` tests catch most violations.

### Dimensional rescaling

Every real argument should carry a dimensional scale factor through
`MOM_unit_scaling.F90`. A new constant `0.5 [s-1]` is wrong; write
`0.5 * US%s_to_T**(-1)`. The `.testing/test.dim` test catches new
unscaled constants.

### Doxygen comments

```fortran
!> Compute the diapycnal diffusivity from KPP.
!! This module wraps the CVMix KPP implementation and adds layer-mode
!! support.
module MOM_CVMix_KPP

  type, public :: KPP_CS  ;  private
    real :: Ri_crit  !< Critical bulk Richardson number [nondim]
    real :: minOBLdepth  !< Minimum allowed OBL depth [Z ~> m]
  end type KPP_CS

contains

!> Initialize the KPP control structure.
subroutine KPP_init(CS, paramFile, ...)
  type(KPP_CS),    intent(inout) :: CS         !< KPP control structure
  type(param_file_type), intent(in) :: paramFile  !< MOM_input handle
  ...
end subroutine KPP_init
```

Inline arg comments use `!<`; module/type comments use `!>`. Keep
descriptions concise. Math goes in `\f$ ... \f$` (inline) or `\f[ ... \f]`
(display) blocks.

---

## Test locally

Before pushing, run the regression suite:

```bash
cd .testing
make -j
make -j test
```

A clean run shows `PASS` for every entry. If you added a new
parameter, also confirm `MOM_parameter_doc.all` regenerates without
unexpected churn:

```bash
cd .testing/work/symmetric/tc1
diff -u <(sort MOM_parameter_doc.all) <(sort ../../../../target/work/symmetric/tc1/MOM_parameter_doc.all) | less
```

For a unit-test addition:

```bash
make -j build.unit
make -j run.unit
```

Add a coverage line if relevant:

```bash
make -j DO_COVERAGE=true test
ls build/symmetric/*.gcov
```

---

## Push and open a PR

```bash
git push -u origin feature/my-change
```

Open the PR against `NOAA-GFDL/MOM6:dev/gfdl`. Title and description
expectations:

- Title states the user-visible change (e.g. "Add Bodner23 mixed-layer
  restratification scheme").
- Body links any related issue, discussion, or paper.
- Body lists the regression status: which `.testing` cases were
  exercised; whether `MOM_parameter_doc` churned and why.
- If new diagnostics were added, list their names.

PR labels are applied by reviewers. Common ones:

- `dev/gfdl` (this is the target branch)
- `do not merge` (work in progress)
- `enhancement`, `bug`, `documentation`

---

## CI: GitHub Actions and GFDL GitLab

When you open the PR, GitHub Actions launches automatically:

- Build with multiple compilers (gfortran, optionally Intel).
- Run `.testing` regression suite.
- Run unit tests.
- Coverage upload to codecov.io.
- Lint: source-style checks.

In parallel, GFDL maintainers run a GitLab CI mirror that exercises the
larger MOM6-examples regression set on GFDL's HPC. This pipeline checks
that production configurations (OM4, CM4 baseline, SHiELD) still
reproduce. It is not visible to non-GFDL contributors directly; the
maintainer who reviews your PR will report results in a comment.

---

## Review

Two reviewer approvals are typical for non-trivial changes. Reviewers
look for:

- Code style compliance.
- Reproducibility-discipline violations.
- Backward compatibility: a new optional parameter must default to the
  legacy behavior.
- Documentation: every new parameter has a one-line description; new
  modules have a module-level Doxygen block.
- `MOM_parameter_doc` churn: only the new parameter you intended.
- Tests: a new physics scheme should have at least one `.testing/tc*`
  toggle that exercises it.

Address review comments by pushing more commits to the PR branch. Do
not force-push during review; reviewers want to see the iteration. Once
approved, the maintainer may squash-merge.

---

## After merge

`dev/gfdl` advances. Periodically GFDL cuts a release, fast-forwards
`main` to a tagged commit, and the new tag (e.g. `2025.01`) propagates
out to MOM6-examples and downstream coupled systems (CM4, OM4, CESM,
E3SM, UFS).

If your change requires updates to MOM6-examples (new diag_table,
changed namelist), open a companion PR against
`NOAA-GFDL/MOM6-examples` with the same title prefix. The maintainer
will coordinate the merges.

---

## Common reasons PRs are rejected or delayed

| Reason | Fix |
|--------|-----|
| `.testing/test.repro` fails | An optimization breaks reproducibility. Parenthesize, use `reproducing_sum`, lower `FCFLAGS_REPRO` |
| `.testing/test.dim` fails | Missing dimensional scale factor on a new parameter |
| `MOM_parameter_doc.all` has unexpected churn | Default value of an existing parameter changed silently. Restore the default and add the new behavior under a new switch |
| New parameter has no description | Add a `description=` kwarg to `get_param` |
| New module lacks Doxygen | Add a `!>` block before `module my_mod` |
| Unparenthesized sums in hot path | Parenthesize for bit reproducibility |
| Halo not updated before stencil | Add `pass_var(...)` before the read |
| New diagnostic not registered | Add `register_diag_field` and `post_data` |
| Non-default behavior change | Wrap behind a new switch with the old default |

---

## Resources

- [MOM6 wiki: Code style guide](https://github.com/NOAA-GFDL/MOM6/wiki/Code-style-guide)
- [MOM6 wiki: Developer workflow](https://github.com/NOAA-GFDL/MOM6/wiki/Developer-workflow)
- [MOM6 wiki: Doxygen](https://github.com/NOAA-GFDL/MOM6/wiki/Doxygen)
- [MOM6 wiki: Run-time parameter system](https://github.com/NOAA-GFDL/MOM6/wiki/MOM6-run-time-parameter-system)
- [MOM6 wiki: Diagnostic vertical remapping](https://github.com/NOAA-GFDL/MOM6/wiki/MOM6-diagnostic-vertical-remapping)
- [.testing/README.rst](https://github.com/NOAA-GFDL/MOM6/blob/main/.testing/README.rst)
- [Adcroft et al. 2019, JAMES](https://doi.org/10.1029/2019MS001726)
- [NOAA-GFDL/MOM6 Discussions](https://github.com/NOAA-GFDL/MOM6/discussions)

---

## Where to next

- Style guide details and rules: this page
- The regression suite that gates merge: `getting-started.md` (Step 6)
- Where the parameters you might add live: `architecture.md`
- Debugging during development: `debugging.md`
