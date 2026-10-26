# mpich-build

Fast builds of [MPICH](https://github.com/pmodels/mpich) from a source
checkout: seconds to a couple of minutes instead of the 15 or more of
`autogen.sh && configure && make`.

The core is **mymake**, a set of Perl scripts generated from
[MyDef](https://github.com/hzhou/MyDef) sources (`*.def`). It has two modes:

- **quick** (`-quick`): replaces `configure` entirely. Config headers are
  filled in from the autoheader templates plus built-in defaults and probes,
  and Makefiles are written straight from the `Makefile.am` files. Configure
  takes a few seconds.
- **slow** (default): runs MPICH's real `autoreconf` and `configure`, and
  extracts a simplified Makefile. For when configure itself is what you are
  testing.

Both modes take the external components (hwloc, libfabric, ucx, yaksa, mpl,
pmi, romio, libtool, ...) prebuilt from a **modules tree**, which
`mymake_modules.pl` builds once per host and variant (`cpu`, `gpu` for CUDA,
`hip`) and packs into a tarball.

## Quick start

Needs [MyDef](https://github.com/hzhou/MyDef) (`mydef_page`), Perl, Python 3,
git, gcc/g++/gfortran and autotools. Details are in
[doc/usage.md](doc/usage.md).

```sh
git clone https://github.com/hzhou-muse/mpich-build
cd mpich-build
for f in mymake mymake_config mymake_ld mymake_libtool mymake_test mymake_log mymake_sync_headers; do
    mydef_page $f.def
done

# quick mode needs the autoheader templates, generated once from a local MPICH
# repository ($MPICH_GITHUB or ~/mpich-github; see doc/usage.md)
perl mymake/mymake_sync_headers.pl

# in an MPICH checkout
cd /path/to/mpich
perl /path/to/mpich-build/mymake/mymake_modules.pl --name=cpu      # once; or gpu, hip
perl /path/to/mpich-build/mymake/mymake.pl -quick --prefix=$PWD/_inst
make -j$(nproc) && make install
make hydra-install                    # mpiexec
```

`mymake_modules.pl` writes `../mpich-modules/modules-cpu.tar.gz` next to the
checkout, and `mymake.pl` finds it there. With `--with-cuda`, build the `gpu`
variant first.

## What is in here

| Path | What |
|---|---|
| `mymake.def` | the main script `mymake.pl` and its small helpers |
| `mymake_config.def` | `mymake_config.pl` (config headers) and `mymake_makefile.pl` (Makefiles), used by quick mode |
| `mymake_modules.def` | `mymake_modules.pl`, the modules builder |
| `mymake_sync_headers.def` | generates the autoheader templates for quick mode |
| `mymake_ld.def`, `mymake_libtool.def`, `mymake_test.def`, `mymake_log.def` | link wrapper and installer, libtool fallback, test Makefiles, function-trace logging |
| `macros_common/`, `macros_mymake/` | the shared build logic: option handling, automake parsing, Makefile writing, and the per-component `config_*.def` |
| `doc/` | [usage](doc/usage.md) and [design](doc/design.md) |

The scripts are generated into `mymake/` (not committed).

## Modules

In quick mode, mpl, mpi_h (the generated `mpi.h` headers), pmi, romio, yaksa
(seq, CUDA, HIP, ZE) and hwloc are not configured with autotools at all; they
are configured by the same Perl code that configures MPICH, and build in
seconds. libfabric and ucx still use autotools. A checkout compares its
sources with the tarball's `modules.stamp` and rebuilds an in-tree component
that differs; `mymake_modules.pl --rebuild=<component> [options]` rebuilds one
component with different options.

## Status and limits

- Linux, x86_64, and gcc are what quick mode is written and tested for.
- yaksa's HIP and ZE support and the `hip` variant were ported from configure
  but have not been tested on hardware.
- Quick mode generates only f77 test Makefiles, and its f77 testlist leaves out
  the spawn, io, rma and profile directories. Use slow mode for those.
- A modules tarball is meant for the host that built it.
- The template generator needs a local MPICH repository: `$MPICH_GITHUB`, or
  `~/mpich-github`, `~/work/mpich-github` or `~/projects/mpich-github`.

See [doc/design.md](doc/design.md) for how it works and for open questions.
