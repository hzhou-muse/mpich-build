# Using mpich-build

Paths below use `<mpich-build>` for this repository. Build in an MPICH
checkout of your choice; the config templates come from a separate local MPICH
repository at `~/work/mpich-github` (its `main` branch).

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
git -C ~/work/mpich-github fetch origin main:main   # the script clones the local main
cd <scratch dir>                           # it clones into ./mpich-checkout
perl <mpich-build>/mymake/mymake_sync_headers.pl
cat <mpich-build>/mymake/config_templates/STAMP
```

The repository path `~/work/mpich-github` is fixed in the script for now.
Expect 8 templates. Re-run it when quick mode trips over a configure change in MPICH.

### 3. Get a modules tarball

Build one on the host that will use it; the build scripts are in
hzhou/mpich-modules for now (`build_all.sh`, or the per-module
`build_*.sh` and `build_tarball.sh`). Then point mymake at it:

```sh
export MODTARBALL=/path/to/modules.tar.gz
```

For `--with-cuda`, put `modules-gpu.tar.gz` (built with `--with-cuda`) next
to `modules.tar.gz`; mymake switches to it automatically. Alternatively,
copy a tarball into the MPICH checkout as `modules.tar.gz`, or set
`MODDIR` to an unpacked one.

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
make mpifort && make mpifort-install
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
| `MODTARBALL` | Path of the modules tarball |
| `MODDIR` | Use this unpacked modules tree instead of a tarball |
| `CC`, `CXX`, `FC` | Compilers (default gcc, g++, gfortran) |

## Troubleshooting

| Symptom | Cause |
|---|---|
| `moddir not set` | No modules tarball found: set `MODTARBALL`. |
| `undefined reference to __isoc23_*`, missing `crtbeginS.o` | The tarball was built on a different host; build one here. |
| Unexpanded `PAC_*` macros in a configure script (slow mode) | confdb wasn't copied into that subdirectory. |
| `make: Nothing to be done for 'test'` | Slow mode: use `make testing`. |
| `No rule to make target 'fortran'` | Slow mode: use `make mpifort`. |
| A quick build misses a new MPICH config symbol | Re-run `mymake_sync_headers.pl`, then teach the matching `macros_mymake/config_*.def` the value if needed. |
