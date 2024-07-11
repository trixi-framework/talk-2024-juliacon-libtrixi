# libtrixi: Serving legacy Fortran codes with simulations in Julia

[![License: MIT](https://img.shields.io/badge/License-MIT-success.svg)](https://opensource.org/licenses/MIT)

This is the companion repository for the talk

**Serving legacy Fortran codes with simulations in Julia**
[*Gregor Gassner*](https://www.mi.uni-koeln.de/NumSim/gregor-gassner/),
[*Benedict Geihe*](https://www.mi.uni-koeln.de/NumSim/dr-benedict-geihe/),
[*Michael Schlottke-Lakemper*](https://lakemper.eu)

[Juliacon2024](https://juliacon.org/2024), Eindhoven, the Netherlands, 9th to 13th July 2024

## Abstract
With [libtrixi](https://github.com/trixi-framework/libtrixi) we present a blueprint for connecting established
research codes to modern software packages written in Julia without sacrificing performance. Specifically,
[libtrixi](https://github.com/trixi-framework/libtrixi) provides an API to [Trixi.jl](https://github.com/trixi-framework/Trixi.jl),
a Julia package for adaptive numerical simulations of conservation laws.

## Slides
The slides can be downloaded [here](slides.pdf).

## Reproducibility

The instructions are written based on the supercomputer
[JUWELS](https://www.fz-juelich.de/en/ias/jsc/systems/supercomputers/juwels), operated at
Jülich Supercomputing Centre, on which the results presented on the poster were obtained.

### Getting started

The following standard software packages need to be made available locally before installing
[libtrixi](https://github.com/trixi-framework/libtrixi):
* C compiler with support for C11 or later (e.g., [GCC](https://gcc.gnu.org/) or [Clang](https://clang.llvm.org/))
  (we used GCC v12.3.0)
* Fortran compiler with support for Fortran 2018 or later (e.g., [Gfortran](https://gcc.gnu.org/fortran/))
  (we used Gfortran v12.3.0)
* [CMake](https://cmake.org/)
  (we used cmake v3.26.3)
* MPI (e.g., [Open MPI](https://www.open-mpi.org/) or [MPICH](https://www.mpich.org/))
  (we used Open MPI v4.1.5)
* [HDF5](https://www.hdfgroup.org/solutions/hdf5/)
  (we used parallel HDF5 v1.14.2)
* [Julia](https://julialang.org/downloads/platform/)
  (it is advised to use tarballs from the official website or the `juliaup` manager, we used v1.10.3)
* [Paraview](https://paraview.org) for visualization
  (we used v5.12.0)

The following software packages require manual compilation and installation. To simplify
the installation instructions, we assume that the environment variable `WORKDIR` was set to the
directory, where all the code is downloaded to and compiled, and `PREFIX` was set to an installation
prefix. For example,
```shell
export WORKDIR=$HOME/repro-poster-2024-pasc-libtrixi
export PREFIX=$WORKDIR/install
```


#### t8code

t8code is a meshing library which is used in `Trixi.jl`. The source code can be obtained from the official
[website](https://github.com/DLR-AMR/t8code). Here is an example for compiling and installing it (we used v2.0.0):

```shell
cd $WORKDIR
git clone --branch 'v2.0.0' --depth 1 https://github.com/DLR-AMR/t8code.git
cd t8code
git submodule init
git submodule update
./bootstrap
./configure --prefix="${PREFIX}/t8code" \
            CFLAGS="-Wall -O2" \
            CXXFLAGS="-Wall -O2" \
            --enable-mpi \
            CXX=mpicxx \
            CC=mpicc \
            --enable-static \
            --without-blas \
            --without-lapack
make
make install
```


#### libtrixi

Here is an example for compiling and installing `libtrixi` (we used v0.1.5):

```shell
cd $WORKDIR
git clone --branch 'v0.1.5' --depth 1 https://github.com/trixi-framework/libtrixi.git
cd libtrixi
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=${PREFIX}/libtrixi ..
make
make install
```

A separate Julia project for use with libtrixi has to be set up:

```shell
mkdir ${PREFIX}/libtrixi-julia
cd ${PREFIX}/libtrixi-julia
${PREFIX}/libtrixi/bin/libtrixi-init-julia \
  --t8code-library ${PREFIX}/t8code/lib/libt8.so \
  ${PREFIX}/libtrixi
```

To exactly reproduce the results presented on the poster, the provided [Manifest.toml](Manifest.toml)
can be used to install the same package versions for all Julia dependencies:

```shell
cp Manifest.toml ${PREFIX}/libtrixi-julia/
cd ${PREFIX}/libtrixi-julia
JULIA_DEPOT_PATH=./julia-depot \
  julia --project=. -e 'import Pkg; Pkg.instantiate()'
```

Alternatively all packages can be updated to the most recent version on demand:

```shell
cd ${PREFIX}/libtrixi-julia
JULIA_DEPOT_PATH=./julia-depot \
  julia --project=. -e 'import Pkg; Pkg.update()'
```


#### Trixi2Vtk

[Trixi2Vtk](https://github.com/trixi-framework/Trixi2Vtk.jl)
needs to be installed for later post processing (we used v0.3.15):

```shell
cd ${PREFIX}/libtrixi-julia
JULIA_DEPOT_PATH=./julia-depot \
  julia --project=. -e 'import Pkg; Pkg.add(name="Trixi2Vtk", version="0.3.15")'
```


### Running the baroclinic instability simulation

The julia scripts for setting up simulations (*libelixirs*) can be found in the examples folder:
```shell
export EXAMPLES=$PREFIX/libtrixi/share/libtrixi/LibTrixi.jl/examples
```

For the results on the poster the resolution of the computational mesh was increased. For
this, modify line 128 of
`${EXAMPLES}/libelixir_p4est3d_euler_baroclinic_instability.jl`
such that `initial_refinement_level = 1`.

We used the following SLURM script to launch the simulation on JUWELS:
```shell
#!/bin/bash -x
#SBATCH --nodes=128
#SBATCH --ntasks-per-node=48

module purge
module load GCC
module load OpenMPI
module load HDF5/1.14.2

srun -n "${SLURM_NTASKS}" \
  ${PREFIX}/libtrixi/bin/trixi_controller_simple_f \
  ${PREFIX}/libtrixi-julia \
  ${EXAMPLES}/libelixir_p4est3d_euler_baroclinic_instability.jl
```

Output files will be written to `./out_baroclinic` and need to be post processed:

```shell
cd ${PREFIX}/libtrixi/out_bubble
JULIA_DEPOT_PATH=${PREFIX}/libtrixi-julia/julia-depot \
   julia --project=${PREFIX}/libtrixi-julia -e 'using Trixi2Vtk; trixi2vtk("solution*.h5")'
```

The results can now be viewed using, e.g., paraview.
We used [slice_surface.pvsm](slice_surface.pvsm) to generate our results.


## Authors
This repository was initiated by
[Benedict Geihe](https://www.mi.uni-koeln.de/NumSim/dr-benedict-geihe/)
and [Michael Schlottke-Lakemper](https://lakemper.eu).


## License
The contents of this repository are licensed under the MIT license (see [LICENSE.md](LICENSE.md)).
