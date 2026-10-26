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
2. Find the modules tree (see [Modules](#modules)); a tarball is unpacked
   into `<checkout>/mymake/` once, marked by `mymake/skip_tarball`.
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

## Modules

MPICH builds a set of components besides its own core: the external
submodules hwloc, libfabric and ucx; the in-tree components mpl, pmi, romio
and yaksa (yaksa is no longer an external module); and the helpers libtool
and MyDef. Running their autogen, configure and make in every checkout costs
several minutes, so mymake takes them prebuilt from a **modules tree**.

### Modules tree and tarball

A modules tree holds the configured and built components. A checkout uses one
as follows:

1. If `$MODDIR` is set, use that tree directly (no copy).
2. Otherwise unpack a modules tarball into the checkout's `mymake/` and use
   that. The tarball is `$MODTARBALL` if set, else
   `../mpich-modules/modules-<variant>.tar.gz` next to the checkout, where
   `<variant>` is `gpu` with `--with-cuda` and `cpu` otherwise. (Today: a
   `modules.tar.gz` in the checkout, or `$MODTARBALL` with an automatic
   `-gpu`/`-hip` file name; the `../mpich-modules` default comes with step 1.)

The Makefile keeps rules to build each component in the tree, so make only
rebuilds what changed. Unpacking per checkout (the default) keeps checkouts
independent: a checkout that rebuilds a component (see drift below) does so in
its own copy. A shared `$MODDIR` skips the copy, for when that isn't needed.

Components the tree doesn't provide are built in-tree as before, so older
tarballs keep working.

- **Host-specific by design.** A tarball is built on, and for, the host that
  uses it; it holds compiled objects and configured headers and isn't meant
  to be portable.
- **Drift is accepted by design.** The components in a tarball can lag behind
  a checkout, in the same way the config templates and the quick-mode logic
  can lag behind MPICH's configure. This drift is fairly stable and rarely
  breaks a build; the answer is to make rebuilds cheap, and to detect drift
  cheaply.

### mymake_modules.pl (planned)

`mymake_modules.pl` builds modules trees and tarballs, using quick mode where
it can.

```
perl <mpich-build>/mymake/mymake_modules.pl --name=<variant> [--mpich=<checkout>]
perl <mpich-build>/mymake/mymake_modules.pl --rebuild=<component> [component options]
```

- **Variants are preset.** `mymake_modules.pl` knows a fixed set of variants
  (`cpu`, `gpu` for CUDA, and others such as hip or ze where the host has
  them) and hard-codes their options, filling in host details by detection
  (e.g. the CUDA prefix from `nvcc` or `/usr/local/cuda`). Each component
  takes the variant in its own terms (e.g. yaksa's `--with-cuda`, ucx's
  `--with-cuda`, mpl's GPU support; debug flags differ per component too),
  so this mapping is per-component code, not a set of shared options. There
  is no config file and no saved options: building a variant is just
  `--name=<variant>`.
- **Source.** Components come from an MPICH checkout (`--mpich`, default the
  current directory): in-tree components from their directories, submodules
  at the commits the checkout records (`git ls-tree HEAD modules/`).
- **Output directory.** By default `mpich-modules/` in the parent folder of
  the checkout, e.g. `~/work/pull-requests/mpich-modules/` for checkouts in
  `~/work/pull-requests/`. It holds:

  ```
  mpich-modules/
      <variant>/<component>/    persistent build trees
      modules-<variant>.tar.gz  packed trees, with a STAMP
  ```

  The build trees are kept, so a rebuild only redoes what changed.
- **Component recipes.** Each component declares its source, its method and
  how the variant options map to it:

  | Component | Source | Method |
  |---|---|---|
  | mpl, pmi, romio | in-tree | quick (mymake's existing `config_*` code) |
  | yaksa | in-tree | quick: run `genpup.py`, write `yaksa_config.h` (template `yaksa_config.h.in`), `yaksa.h` and `localdefs`, and the Makefile from `Makefile.am`; seq, cuda, hip and ze backends |
  | hwloc | submodule | quick: `autoheader` for the private config header template, then the config headers, `static-components.h` and the Makefile of `hwloc/` (Linux, embedded, no optional components) |
  | libfabric, ucx | submodules | autotools at first, quick later where it pays off |
  | libtool, mydef_boot | as today | as today |

- **Customization is a single-component rebuild.** Anything beyond the preset
  variants starts from an existing variant tarball: run inside a checkout,
  `--rebuild=<component> [component options]` rebuilds just that component in
  the checkout's modules tree, with the variant's options plus the extra
  ones (e.g. a debug build of yaksa), and leaves the shared tarball alone.
  The same command serves debugging a branch's change to a component.
- **STAMP.** Each tarball records the MPICH commit it was built from, the
  options, the compilers, and per component either the submodule commit or,
  for in-tree components, the git tree hash of its directory
  (`git rev-parse HEAD:src/mpl`). A tree hash changes only when something
  under that directory changes, so one tarball matches every branch that
  leaves the component alone. On the checkout side mymake hashes the working
  tree, so uncommitted edits count (`git add -A <dir>` into a temporary index,
  then `git write-tree --prefix=<dir>/`; ignored generated files don't
  count). At configure time mymake compares them with the checkout:
  - same: use the prebuilt component;
  - different, in-tree component: rebuild it in quick mode from the
    checkout's source, into the checkout's modules tree (seconds for mpl,
    pmi and romio). This matters most for mpl, whose headers come from the
    checkout;
  - different submodule commit: warn.
  It also warns when the build's options don't match the variant's (e.g.
  `--with-cuda` with a cpu tarball).

### Steps

1. mpl, pmi and romio into the modules tree, built with the existing quick
   code; the STAMP with tree hashes; rebuild-on-mismatch; the MODDIR /
   MODTARBALL / `../mpich-modules` lookup.
2. Quick mode for yaksa.
3. hwloc, libfabric and ucx through the same builder (autotools), replacing
   the scripts of the old hzhou/mpich-modules repository, which is then
   retired.
4. Optionally, quick mode for hwloc and libfabric (and ucx).

## Decisions

- Keep `mymake/` as the output directory name.
- Don't commit the config templates; generate them with
  `mymake_sync_headers.pl` at setup and when a discrepancy is noticed.
- Modules tarballs are host-specific, and drift is accepted.
- A checkout finds its modules through `$MODDIR` (used directly), else
  `$MODTARBALL`, else `../mpich-modules/modules-<variant>.tar.gz` (variant
  `gpu` with `--with-cuda`, else `cpu`), unpacked into the checkout.
- On a tree-hash mismatch, an in-tree component is rebuilt automatically.
- Variants are preset in `mymake_modules.pl`, with options hard-coded per
  component and host details detected; no config file, no saved options.
  Further customization means rebuilding individual components on top of an
  existing variant tarball.
- `mymake_modules.pl` writes to `mpich-modules/` next to the checkout by
  default. The old hzhou/mpich-modules repository will be retired, which
  frees the name.

## Open questions

- Which variants to preset: `cpu` and `gpu` (CUDA) for sure; hip and ze only
  when detected? Debug builds of the whole set as a variant, or only through
  single-component rebuilds?
- Should a customized checkout tree be packable as its own tarball, or stay
  checkout-local?

## Planned

- **Cheap drift detection for the config templates**: compare the template
  STAMP with the checkout's `configure.ac` files.
- **Gaps between the modes**: the Fortran targets are `fortran` in quick
  mode and `mpifort` in slow mode; quick mode generates only f77 test
  Makefiles, and its f77 testlist drops the spawn, io, rma and profile
  directories.
- **Robustness**: many `system` calls ignore failures, so a missing tool can
  fail silently.
