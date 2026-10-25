# mpich-build design

mpich-build builds MPICH from a source checkout faster than the stock
`autogen.sh && configure && make`. Its core is **mymake**, a set of Perl
scripts generated from MyDef sources (`*.def`).

## Goals

- Fast day-to-day builds of MPICH branches, in seconds to a couple of
  minutes instead of 15 or more.
- A full-fidelity mode for when configure itself matters.
- Easy customization: the build logic is plain Perl that we own, not
  autoconf output.

Non-goals: replacing MPICH's official build system, building release
tarballs, CI orchestration (the Jenkins tooling stays in mpich_study), and
portable binary artifacts.

## Layout

| Path | What |
|---|---|
| `mymake.def` | Main page `mymake.pl`, plus the small helper pages `mymake_{errmsg,cvars,logs,hydra,stage,clean}.pl` |
| `mymake_config.def` | `mymake_config.pl` (writes config headers) and `mymake_makefile.pl` (writes Makefiles), used by quick mode |
| `mymake_ld.def` | `mymake_ld.pl` (link wrapper) and `mymake_install.pl` |
| `mymake_libtool.def` | `mymake_libtool.pl`, a fallback that builds a libtool script when the modules tarball has none |
| `mymake_test.def` | `mymake_test.pl`, and `mymake_testing.pl` for slow-mode test setup |
| `mymake_log.def` | `mymake_log.pl` / `mymake_log_show.pl`: add function-trace logging to sources and display it (debugging aid) |
| `mymake_sync_headers.def` | `mymake_sync_headers.pl`: generates the quick-mode config templates |
| `macros_common/` | Generic helpers: options, file I/O, automake parsing (`load_automake`), Makefile writing (`dump_makefile`), config header writing, libtool |
| `macros_mymake/` | MPICH specifics: arguments, subdirectories (mpl, pmi, romio, hydra), the per-component `config_*.def` used by quick mode, bindings, wrappers (`mpicc_etc.def`) |
| `mymake/` | Generated output (not committed): the `*.pl` scripts and `config_templates/` |

Every page has `output_dir: ./mymake`, so all scripts land in `mymake/`, and
`mymake.pl` finds its helpers next to itself (`$opts{mymake}_<name>.pl`).

Inside an MPICH checkout, mymake also uses a directory named `mymake/` for its
state: the saved arguments (`mymake/args`, `mymake/opts`), the unpacked
modules, and intermediate files.

## Build flow

Common to both modes (`mymake.pl`):

1. Parse arguments. With no arguments, reuse the last ones from
   `mymake/args`. With new arguments and an existing Makefile, run
   `make realclean` first unless `-noclean` is given.
2. Unpack the modules tarball into `<checkout>/mymake/` (once; marked by
   `mymake/skip_tarball`).
3. Run MPICH's own generators: `gen_binding_c.py`, `gen_abi.py`,
   `gen_ch4_api.py`, `gen_coll.py`, `gen_pmi_msg.py`, `mpicc.def` (MyDef), the
   C++ and Fortran binding generators. These are what `autogen.sh` would run,
   minus autoconf.

### Quick mode (`-quick`)

Replaces configure entirely:

- **libtool**: copied from the tarball's `libtool/libtool` and patched
  (e.g. the CXX tag no longer hard-codes a g++ version's crt files).
- **Config headers**: `mymake_config.pl <component>` reads the autoheader
  template for the component from `mymake/config_templates/`, fills in
  values from built-in defaults, from probes (type sizes, available headers,
  optional types such as `__float128`, Fortran kinds), and from the
  arguments, and writes the header (e.g. `src/include/mpichconf.h`).
- **Makefiles**: `mymake_makefile.pl <component>` parses the component's
  `Makefile.am` with the automake conditionals mymake decides, and writes a
  flat Makefile with explicit rules.
- **Subcomponents** (mpl, pmi, romio, hydra, Fortran binding, test suite,
  dtpools) are configured lazily by Makefile rules of the form
  `$(DO_config) X && $(DO_makefile) X`, the first time a target needs them.

### Slow mode (default)

Uses MPICH's real configure:

- Edits `configure.ac` to drop `AC_CONFIG_SUBDIRS`, runs `libtoolize &&
  autoreconf`, then `./configure --with-pm=no <args>`.
- Moves the generated Makefile to `mymake/Makefile.orig` and extracts a
  simplified Makefile from it.
- mpl, pmi and romio are autoreconf'd and configured by Makefile rules;
  hydra is configured and built separately; the Fortran binding runs its own
  `autogen.sh` and `configure` (`make mpifort`).
- The test suite is set up by `make testing`, which runs `test/mpi`'s
  autogen and configure.

## Config templates

Quick mode needs the autoheader templates (`*.h.in`) of MPICH's configure
scripts: mpichconf, mplconfig, pmi_config, romioconf, hydra_config,
mpitestconf, dtpoolsconf and mpifort_config. They are not committed.
`mymake_sync_headers.pl` clones `main` of a local MPICH repository, runs
aclocal and autoheader for each component, and copies the results into
`mymake/config_templates/`. `STAMP` records the MPICH commit and autoconf
version they came from.

Templates are generated at setup and re-synced lazily, only when a
difference shows up (a new `#undef` that quick mode doesn't know about, a
build problem traced to a stale template).

## Modules tarball

MPICH depends on hwloc, yaksa, libfabric and ucx (and uses libtool and
MyDef). Running their autogen, configure and make for every checkout costs
several minutes. mymake takes them from a **modules tarball**: configured
and already built trees, unpacked into each checkout's `mymake/`. The Makefile
still has rules to build them, so make only rebuilds what changed.

- **Host-specific by design.** A tarball is built on, and for, the host
  that uses it; it holds compiled objects and configured headers and isn't
  meant to be portable.
- **Located through `$MODTARBALL`** (or `modules.tar.gz` in the checkout, or
  `$MODDIR` pointing to an unpacked tree). With `--with-cuda` or `--with-hip`
  mymake uses `modules-gpu.tar.gz` / `modules-hip.tar.gz` next to the given
  file, if it exists.
- **Drift is accepted by design.** The module versions in a tarball can lag
  behind an MPICH branch's submodules, in the same way the config templates
  and the quick-mode logic can lag behind MPICH's configure. This drift is
  fairly stable and rarely breaks a build; the answer is to make rebuilds
  cheap, not to rebuild on every change.

## Decisions

- Keep `mymake/` as the output directory name.
- Don't commit the config templates; generate them with
  `mymake_sync_headers.pl` at setup and when a discrepancy is noticed.
- `$MODTARBALL` is the way to locate the modules tarball; there is no
  default location.
- Tarballs are host-specific, and drift is accepted.

## Planned

- **Module builds in mpich-build.** Bring in the module build scripts
  (today in hzhou/mpich-modules) so one repository covers the whole setup.
- **Quick mode for modules**, starting with yaksa. yaksa is no longer an
  external module in MPICH (it lives in `src/mpi/datatype/typerep/yaksa`), so
  it needs a quick mode anyway: run its generators (`genpup.py`), write
  `yaksa_config.h` from a template and its Makefile from `Makefile.am`, as
  for MPICH. The same approach would then cover hwloc, libfabric and ucx,
  making module rebuilds cheap and customizable.
- **Cheap drift detection** (TODO): a STAMP in the modules tarball (MPICH and
  module commits, compiler, options) that mymake compares with the checkout,
  and a similar check of the template STAMP.
- **Gaps between the modes**: the Fortran targets are `fortran` in quick
  mode and `mpifort` in slow mode; quick mode generates only f77 test
  Makefiles, and its f77 testlist drops the spawn, io, rma and profile
  directories.
- **Robustness**: many `system` calls ignore failures, so a missing tool can
  fail silently.
