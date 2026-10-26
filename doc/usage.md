# Using mpich-build

Paths below use `<mpich-build>` for this repository. Build in an MPICH
checkout of your choice; the config templates come from a separate local MPICH
repository (its `main` branch): `$MPICH_GITHUB`, or else the first of
`~/mpich-github`, `~/work/mpich-github` and `~/projects/mpich-github`.

## Prerequisites

- MyDef (`mydef_page` on `PATH`) to compile the pages.
- Perl, Python 3, git.
- gcc and g++; gfortran for the Fortran binding and tests (otherwise build
  with `--disable-fortran`).
- Autotools (autoconf ≥ 2.69, automake ≥ 1.15, libtool ≥ 2.4.4) for slow
  mode, for generating the config templates, and for building the modules
  tarball. Quick builds themselves don't run autotools.
- Optional: CUDA (`--with-cuda`) with the GPU modules tarball.

## Setup

### 1. Compile the pages

```sh
cd <mpich-build>
for f in mymake mymake_config mymake_ld mymake_libtool mymake_test mymake_log mymake_sync_headers; do
    mydef_page $f.def
done
```

The scripts land in `<mpich-build>/mymake/`. Check the output for
`Code ... not found!` or `... .def not found`: mydef still writes the script,
with that part missing. Recompile after pulling or editing; the `macros_*`
files are shared between pages, so recompiling all of them is simplest.

### 2. Generate the config templates (quick mode)

```sh
git -C <mpich repo> fetch origin main:main   # the script clones the local main
cd <scratch dir>                           # it clones into ./mpich-checkout
perl <mpich-build>/mymake/mymake_sync_headers.pl
cat <mpich-build>/mymake/config_templates/STAMP
```

The script takes the MPICH repository from `MPICH_GITHUB`; if that is unset it
uses the first of `~/mpich-github`, `~/work/mpich-github` and
`~/projects/mpich-github` that exists, and stops if there is none.
Expect 10 templates (it fetches the hwloc submodule for `hwloc_config.h`, so it
needs network). Re-run it when quick mode trips over a configure change in
MPICH, or when a build says a template is stale.

### 3. Get a modules tarball

The modules (libtool, hwloc, yaksa, libfabric, ucx, mydef_boot, and the quick
builds of mpl, mpi_h, pmi and romio) are built once per host and variant, from
an MPICH checkout, at the commits that checkout records:

```sh
cd <MPICH checkout>
perl <mpich-build>/mymake/mymake_modules.pl --name=cpu     # or gpu (CUDA), hip
```

It writes `../mpich-modules/modules-<variant>.tar.gz` next to the checkout,
keeping the build trees in `../mpich-modules/<variant>/` so a rerun only
redoes what changed (no-op rerun: seconds; gpu from scratch: a few minutes,
ucx is the slowest). Options: `--mpich=<checkout>`, `--out=<dir>`, `-j=N`,
`--only=a,b` (rebuild just these), `--force`. The `gpu` variant needs `nvcc` on
`PATH` (or `/usr/local/cuda`), `hip` needs ROCm in `/opt/rocm`.

A checkout finds its modules in this order: `$MODDIR` (an unpacked tree, used
in place), the tree already unpacked in the checkout's `mymake/`,
`$MODTARBALL`, `modules.tar.gz` in the checkout, then
`../mpich-modules/modules-<variant>.tar.gz` (variant `gpu` with `--with-cuda`,
`hip` with `--with-hip`, else `cpu`). A `modules[-<name>].tar.gz` that is given
is swapped for `modules-<variant>.tar.gz` next to it when that exists.

Each tarball carries a `modules.stamp` (variant, MPICH commit, compilers, and a
key per component). On every configure mymake compares it with the checkout:
it warns when the variant or a submodule commit differs, and rebuilds an
in-tree component (yaksa, mpl, mpi_h, pmi, romio) that differs, in the
checkout's own copy (seconds; yaksa about 30 s). To change a component's
options, rebuild it in the checkout:

```sh
perl <mpich-build>/mymake/mymake_modules.pl --rebuild=yaksa CFLAGS=-O1
```

(`CFLAGS=` replaces the default `-g -O2`; other arguments are configure
options.)

## Quick build

From the MPICH checkout:

```sh
perl <mpich-build>/mymake/mymake.pl -quick --prefix=$PWD/_inst [configure options]
make -j$(nproc) && make install
make hydra-install                # mpiexec / mpirun
make fortran && make fortran-install
```

Common options: `--with-device=ch4:ofi` (default) or `ch4:ucx`,
`--enable-g=dbg`, `--enable-fast=O3,ndebug`, `--with-ch4-max-vcis=N`,
`--with-cuda=<path>`, `--disable-fortran`, `--disable-cxx`,
`--disable-romio`, `--enable-mpi-abi`.

Quick configure takes a few seconds. A fresh clone needs no
`git submodule update`; the modules come from the tarball.

## Slow build

Same commands without `-quick`; it runs autoreconf and MPICH's configure:

```sh
perl <mpich-build>/mymake/mymake.pl --prefix=$PWD/_inst [configure options]
make -j$(nproc) && make install
make hydra-install
make fortran && make fortran-install
```

Use it when the configure step itself is what you are testing.

## Re-running mymake

- `perl mymake.pl` with no arguments reuses the last arguments
  (`mymake/args`).
- With new arguments and an existing Makefile, mymake runs
  `make realclean` first; `-noclean` skips that.
- After changing only `.c` files: `make && make install`.
- After changing headers: `make clean`, then rebuild (the generated
  Makefiles don't track header dependencies).
- After changing a `Makefile.am` or `configure.ac`: re-run mymake.
- Stale or interrupted state: `make realclean`, then re-run mymake. If
  there is no Makefile yet, start from a fresh clone.
- In quick mode a single component's Makefile can be regenerated with
  `perl <mpich-build>/mymake/mymake_makefile.pl <component>`
  (`mpich`, `mpl`, `pmi`, `romio`, `hydra`, `fortran`, `test`, `dtpools`).

Other subcommands: `perl mymake.pl clean|errmsg|cvars|logs|hydra|testing|test|makefile|config|libtool`
run the matching helper script.

## Tests

```sh
export PATH=$PWD/_inst/bin:$PATH
make test        # quick mode: mymake writes the test Makefiles (C and f77)
make testing     # slow mode: runs test/mpi's autogen and configure
cd test/mpi/coll
make bcasttest && mpirun -n 4 ./bcasttest
```

- Run tests individually with `mpirun -n <np> ./<test> <args>`, taking the
  arguments from the directory's `testlist` (e.g. `allred` needs
  `-count=10`); GPU variants are in `testlist.gpu`.
- `MPIEXEC_TIMEOUT=<seconds>` bounds a run; `MPIR_CVAR_ODD_EVEN_CLIQUES=1`
  splits one host into fake nodes so the netmod is exercised.
- Programs listed as `EXTRA_PROGRAMS` (e.g. `allocmemf`) are not built by
  default; build them with `make <name>`.
- Quick mode has no f90/f08 test Makefiles, and its f77 testlist leaves out
  the spawn, io, rma and profile directories. Use slow mode for those.

## Environment variables

| Variable | Meaning |
|---|---|
| `MODTARBALL` | Path of the modules tarball (default `../mpich-modules/modules-<variant>.tar.gz`) |
| `MODDIR` | Use this unpacked modules tree instead of a tarball (not rebuilt on drift) |
| `CC`, `CXX`, `FC` | Compilers (default gcc, g++, gfortran) |

## Troubleshooting

| Symptom | Cause |
|---|---|
| `no modules tarball` | None of the places above has one: run `mymake_modules.pl`, or set `MODTARBALL`. |
| `undefined reference to __isoc23_*`, missing `crtbeginS.o` | The tarball was built on a different host; build one here. |
| CUDA MPICH linked against CPU modules | Variant mismatch (a warning is printed): use the `gpu` tarball. |
| Unexpanded `PAC_*` macros in a configure script (slow mode) | confdb wasn't copied into that subdirectory. |
| `make: Nothing to be done for 'test'` | Slow mode: use `make testing`. |
| A quick build misses a new MPICH config symbol | Re-run `mymake_sync_headers.pl`, then teach the matching `macros_mymake/config_*.def` the value if needed. |
