# Introduction

This document provides instructions on how to build CP2K 2023.1.xsmm and its dependencies on ARCHER2 full system.

Please note, this CP2K version includes the `libxsmm` library, which supports small matrix-matrix multiplications,
and replaces CP2K's own `libsmm` library. The `libxsmm` library is not included in the default CP2K 2023.1 build due
to the fact that CP2K with `libxsmm` is known to fail on ARCHER2 for some hybrid MPI/OpenMP configurations. For other
types of job however, this version may give a faster time to solution compared to CP2K 2023.1.

Further information on CP2K can be found at [the CP2K website](https://www.cp2k.org) and on the
[ARCHER2 CP2K documentation page](https://docs.archer2.ac.uk/research-software/cp2k/).

The official build instructions for CP2K are at [https://github.com/cp2k/cp2k/blob/master/INSTALL.md](https://github.com/cp2k/cp2k/blob/master/INSTALL.md).
The ARCHER2 build instructions however use the "manual" route, building each relevant prerequisite independently.

These instructions are also provided as a Slurm batch script, see the `submit.ll` file at [https://github.com/hpc-uk/build-instructions/tree/main/apps/CP2K/ARCHER2-CP2K-2023.1.xsmm](https://github.com/hpc-uk/build-instructions/tree/main/apps/CP2K/ARCHER2-CP2K-2023.1.xsmm).

## General

* We will use the GNU programming environment.
* We will only consider psmp build for CP2K.
* The autotuned version of libgrid was not built as there were some
  residual problems with the automatic code generation on ARCHER2.


# Preliminaries

## Specify versions of CP2K and supporting libraries

```
CP2K_VERSION=2023.1
LIBINT_VERSION=2.6.0
LIBINT_VERSION_SUFFIX=cp2k-lmax-4
LIBXC_VERSION=6.1.0
LIBXSMM_VERSION=1.17
ELPA_VERSION=2022.11.001
PLUMED_VERSION=2.8.2
```

## Load modules

```
module load PrgEnv-gnu
module load cray-fftw
module load cray-python
module load mkl
```

The above commands load the GNU compiler suite, `cray-fftw` and `cray-python`.
Then, the Intel Maths Kernel Library (MKL) module is loaded, replacing the `cray-libsci` module.


## Prepare CP2K build environment

```
export FCFLAGS="-fallow-argument-mismatch"

PRFX=/path/to/work/dir # e.g., /work/y07/shared/apps/core 
CP2K_LABEL=cp2k
CP2K_NAME=${CP2K_LABEL}-${CP2K_VERSION}
CP2K_BASE=${PRFX}/${CP2K_LABEL}
CP2K_ROOT=${CP2K_BASE}/${CP2K_NAME}.xsmm

mkdir -p ${CP2K_BASE}
cd ${CP2K_BASE}

rm -rf ${CP2K_NAME}.xsmm

mkdir tmp
cd tmp

wget -q https://github.com/${CP2K_LABEL}/${CP2K_LABEL}/releases/download/v${CP2K_VERSION}/${CP2K_NAME}.tar.bz2
bunzip2 ${CP2K_NAME}.tar.bz2
tar xf ${CP2K_NAME}.tar
rm ${CP2K_NAME}.tar
mv ${CP2K_NAME} ../${CP2K_NAME}.xsmm

cd ..
rmdir tmp

mkdir ${CP2K_ROOT}/libs
```

## Prepare CP2K arch file

Download the `ARCHER2.psmp` file from [https://github.com/hpc-uk/build-instructions/tree/main/apps/CP2K/ARCHER2-CP2K-2023.1.xsmm](https://github.com/hpc-uk/build-instructions/tree/main/apps/CP2K/ARCHER2-CP2K-2023.1.xsmm)
and copy to `${PRFX}/${CP2K_LABEL}/${CP2K_NAME}/arch/`.

```
sed -i "s:<CP2K_ROOT>:${CP2K_ROOT}:" ${CP2K_ROOT}/arch/ARCHER2.psmp
sed -i "s:<LIBINT_VERSION>:${LIBINT_VERSION}:" ${CP2K_ROOT}/arch/ARCHER2.psmp
sed -i "s:<LIBINT_VERSION_SUFFIX>:${LIBINT_VERSION_SUFFIX}:" ${CP2K_ROOT}/arch/ARCHER2.psmp
sed -i "s:<LIBXC_VERSION>:${LIBXC_VERSION}:" ${CP2K_ROOT}/arch/ARCHER2.psmp
sed -i "s:<LIBXSMM_VERSION>:${LIBXSMM_VERSION}:" ${CP2K_ROOT}/arch/ARCHER2.psmp
sed -i "s:<ELPA_VERSION>:${ELPA_VERSION}:" ${CP2K_ROOT}/arch/ARCHER2.psmp
sed -i "s:<PLUMED_VERSION>:${PLUMED_VERSION}:" ${CP2K_ROOT}/arch/ARCHER2.psmp
```


## Build libint

CP2K releases versions of `libint` appropriate for CP2K at https://github.com/cp2k/libint-cp2k .
A choice is required on the highest `lmax` supported: we choose `lmax = 4` to limit the size of the static executable.

```
cd ${CP2K_ROOT}/libs

LIBINT_LABEL=libint
LIBINT_NAME=${LIBINT_LABEL}-${LIBINT_VERSION}
LIBINT_ARCHIVE=${LIBINT_LABEL}-v${LIBINT_VERSION}-${LIBINT_VERSION_SUFFIX}
LIBINT_ROOT=${CP2K_ROOT}/libs/${LIBINT_LABEL}

rm -rf ${LIBINT_ROOT}
mkdir -p ${LIBINT_ROOT}
cd ${LIBINT_ROOT}

mkdir ${LIBINT_NAME}
cd ${LIBINT_NAME}

wget -q https://github.com/${CP2K_LABEL}/${LIBINT_LABEL}-${CP2K_LABEL}/releases/download/v${LIBINT_VERSION}/${LIBINT_ARCHIVE}.tgz
tar zxf ${LIBINT_ARCHIVE}.tgz
rm ${LIBINT_ARCHIVE}.tgz
mv ${LIBINT_ARCHIVE} ${LIBINT_VERSION_SUFFIX}
cd ${LIBINT_VERSION_SUFFIX}

CC=cc CXX=CC FC=ftn LDFLAGS=-dynamic ./configure \
    --enable-fortran --with-cxx-optflags=-O \
    --prefix=${LIBINT_ROOT}/${LIBINT_VERSION}/${LIBINT_VERSION_SUFFIX}

make -j 8
make -j 8 install
make -j 8 clean
```


## Build libxc

```
cd ${CP2K_ROOT}/libs

LIBXC_LABEL=libxc
LIBXC_NAME=${LIBXC_LABEL}-${LIBXC_VERSION}
LIBXC_ROOT=${CP2K_ROOT}/libs/${LIBXC_LABEL}

rm -rf ${LIBXC_ROOT}
mkdir -p ${LIBXC_ROOT}
cd ${LIBXC_ROOT}

wget -q https://gitlab.com/${LIBXC_LABEL}/${LIBXC_LABEL}/-/archive/${LIBXC_VERSION}/${LIBXC_NAME}.tar.gz
tar zxf ${LIBXC_NAME}.tar.gz
rm ${LIBXC_NAME}.tar.gz
cd ${LIBXC_NAME}

autoreconf -i

CC=cc CXX=CC FC=ftn ./configure --prefix=${LIBXC_ROOT}/${LIBXC_VERSION}

make -j 8
make -j 8 install
make -j 8 clean
```


# Build libxsmm

```
cd ${CP2K_ROOT}/libs

LIBXSMM_LABEL=libxsmm
LIBXSMM_NAME=${LIBXSMM_LABEL}-${LIBXSMM_VERSION}
LIBXSMM_ROOT=${CP2K_ROOT}/libs/${LIBXSMM_LABEL}

rm -rf ${LIBXSMM_ROOT}
mkdir -p ${LIBXSMM_ROOT}
cd ${LIBXSMM_ROOT}

wget -q https://github.com/${LIBXSMM_LABEL}/${LIBXSMM_LABEL}/archive/refs/tags/${LIBXSMM_VERSION}.tar.gz
tar zxf ${LIBXSMM_VERSION}.tar.gz
rm ${LIBXSMM_VERSION}.tar.gz
cd ${LIBXSMM_NAME}

make CC=cc CXX=CC FC=ftn INTRINSICS=1 PREFIX=${LIBXSMM_ROOT}/${LIBXSMM_VERSION} install
```


## Build ELPA

```
cd ${CP2K_ROOT}/libs

ELPA_LABEL=elpa
ELPA_NAME=${ELPA_LABEL}-${ELPA_VERSION}
ELPA_ROOT=${CP2K_ROOT}/libs/${ELPA_LABEL}

rm -rf ${ELPA_ROOT}
mkdir -p ${ELPA_ROOT}
cd ${ELPA_ROOT}

wget -q https://elpa.mpcdf.mpg.de/software/tarball-archive/Releases/${ELPA_VERSION}/${ELPA_NAME}.tar.gz
tar zxf ${ELPA_NAME}.tar.gz
rm ${ELPA_NAME}.tar.gz

cd ${ELPA_ROOT}/${ELPA_NAME}
mkdir build-serial
cd build-serial

export LIBS="-L${MKLROOT}/lib/intel64 -lmkl_scalapack_lp64 -Wl,--no-as-needed -lmkl_gf_lp64 -lmkl_gnu_thread -lmkl_core -lmkl_blacs_intelmpi_lp64 -lgomp -lpthread -lm -ldl"

CC=cc CXX=CC FC=ftn LDFLAGS=-dynamic ../configure       \
  --enable-openmp=no --enable-shared=no \
  --disable-avx512 --disable-detect-mpi-launcher \
  --without-threading-support-check-during-build \
  --prefix=${ELPA_ROOT}/${ELPA_VERSION}/serial

make -j 8
make -j 8 install
make -j 8 clean

cd ${ELPA_ROOT}/${ELPA_NAME}
mkdir build-openmp
cd build-openmp

export LIBS="-L${MKLROOT}/lib/intel64 -lmkl_scalapack_lp64 -Wl,--no-as-needed -lmkl_gf_lp64 -lmkl_gnu_thread -lmkl_core -lmkl_blacs_intelmpi_lp64 -lgomp -lpthread -lm -ldl"

CC=cc CXX=CC FC=ftn LDFLAGS=-dynamic ../configure \
  --enable-openmp=yes --enable-shared=no --enable-allow-thread-limiting \
  --disable-avx512 --disable-detect-mpi-launcher \
  --without-threading-support-check-during-build \
  --prefix=${ELPA_ROOT}/${ELPA_VERSION}/openmp

make -j 8
make -j 8 install
make -j 8 clean
```


## Build Plumed

```
cd ${CP2K_ROOT}/libs

PLUMED_LABEL=plumed
PLUMED_NAME=${PLUMED_LABEL}-${PLUMED_VERSION}
PLUMED_ROOT=${CP2K_ROOT}/libs/${PLUMED_LABEL}

rm -rf ${PLUMED_ROOT}
mkdir -p ${PLUMED_ROOT}
cd ${PLUMED_ROOT}

wget -q https://github.com/${PLUMED_LABEL}/${PLUMED_LABEL}2/archive/refs/tags/v${PLUMED_VERSION}.tar.gz
tar zxf v${PLUMED_VERSION}.tar.gz
rm v${PLUMED_VERSION}.tar.gz
mv ${PLUMED_LABEL}2-${PLUMED_VERSION} ${PLUMED_NAME}
cd ${PLUMED_NAME}

CC=cc CXX=CC FC=ftn MPIEXEC=srun ./configure \
  --disable-openmp --disable-shared --disable-dlopen \
  --prefix=${PLUMED_ROOT}/${PLUMED_VERSION}

make -j 8
make -j 8 install
make -j 8 clean
```


## Build CP2K

```
cd ${CP2K_ROOT}

make -j 8 ARCH=ARCHER2 VERSION=psmp
make -j 8 clean ARCH=ARCHER2 VERSION=psmp
```


### Regression tests

Download the `ARCHER2-regtest.psmp.conf` configuration file from [https://github.com/hpc-uk/build-instructions/tree/main/apps/CP2K/ARCHER2-CP2K-2023.1.xsmm](https://github.com/hpc-uk/build-instructions/tree/main/apps/CP2K/ARCHER2-CP2K-2023.1.xsmm)
and copy to `${CP2K_ROOT}`.

The test can be executed in the queue system by submitting script from `${CP2K_ROOT}`.

```
#!/bin/bash

#SBATCH --job-name=regtest
#SBATCH --time=04:00:00
#SBATCH --exclusive
#SBATCH --partition=standard
#SBATCH --qos=standard
#SBATCH --account=<budget code>
#SBATCH --nodes=1
#SBATCH --ntasks=2
#SBATCH --tasks-per-node=2
#SBATCH --cpus-per-task=2

module load PrgEnv-gnu
module load cray-fftw
module load cray-python
module load mkl

export OMP_NUM_THREADS=2

export SRUN_CPUS_PER_TASK=$SLURM_CPUS_PER_TASK

./tools/regtesting/do_regtest -nobuild -c ./ARCHER2-regtest.psmp.conf
```

The regression tests should take around 3-4 hrs to complete (assuming CPU frequency of 2.0 GHz)
and the summarized results should be as shown below.

```
--------------------------------- Summary --------------------------------
Number of FAILED  tests 0
Number of WRONG   tests 0
Number of CORRECT tests 3797
Total number of   tests 3797
GREPME 0 0 3797 0 3797 X

Summary: correct: 3797 / 3797; 160min
Status: OK
```


## Performance

### H2O

The CP2K H2O benchmark, `QS_DM_LS/H2O-dft-ls.NREP4.inp`, was run over four nodes with
16 MPI tasks per node and 8 OpenMP threads per task.

The runtime and energy results, dated 2023-07-06, were obtained via the `sacct` command.

```bash
sacct -j <job id> --format=JobID,JobName%20,NNodes,ReqCPUFreq,Elapsed,ConsumedEnergyRaw
```

The actual numbers used were the highest runtimes and energies indicated in the `sacct` output.

#### Runtime (s)

Comms | Turbo | Cnt | Min | Max | Avg
----- | ----- | --- | --- | --- | ---
  OFI | &cross; |   3 | 266 | 272 | 270 
  OFI | &check; |   3 | 223 | 228 | 226 

#### Energy [J]

Comms | Turbo | Cnt |     Min |     Max |     Avg
----- | ----- | --- | ------- | ------- | -------
  OFI | &cross; |   3 | 392,627 | 397,917 | 392,473
  OFI | &check; |   3 | 417,549 | 421,330 | 419,765


### Lithium Hydride crystal

The `QS_LiH_HFX/input_bulk_HFX_3.inpz` benchmark was run over 96 nodes with 16 MPI tasks
per node and 8 OpenMP threads per task.

The runtime and energy results, dated 2023-07-07, were obtained via the `sacct` command.

```bash
sacct -j <job id> --format=JobID,JobName%20,NNodes,ReqCPUFreq,Elapsed,ConsumedEnergyRaw
```

The actual numbers used were the highest runtimes and energies indicated in the `sacct` output.

#### Runtime (s)

Comms | Turbo | Cnt | Min | Max | Avg
----- | ----- | --- | --- | --- | ---
  OFI | &cross; |   3 | 144 | 147 | 145
  OFI | &check; |   3 | 119 | 122 | 121 

#### Energy [J]

Comms | Turbo | Cnt |       Min |       Max |       Avg
----- | ----- | --- | --------- | --------- | ---------
  OFI | &cross; |   3 | 4,389,456 | 4,396,607 | 4,374,493
  OFI | &check; |   3 | 4,604,368 | 4,680,678 | 4,643,515 
