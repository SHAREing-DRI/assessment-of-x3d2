---
License: Copyright © 2026 Durham University, SHAREing Project, MIT Licensed
Creator: Andrew Naden
Contributors: Andrew Naden; Ananya Gangopadhyay
Summary: Pre-assessment report template document
---

# <img src='./images/logo.svg' width=90 style="vertical-align:middle" alt="SHAREing logo"/> SHAREing: Pre-assessment report of x3d2

An external submission of x3d2 ([github](https://github.com/xcompact3d/x3d2/), commit 84c750f38af041661852dc35858c879293146db6 `Faster OpenMP Reorder and Accumulation Kernels (#329)`) was made on the 8th of July 2026 to the SHAREing team at Durham.

The pre-assessment of the CPU configuration was performed by Emily Wilkinson of Durham University between the 8th of July 2026 and the 13th of August 2026.

Due to compilation errors arising for the GPU configuration with the submitted version, [commit fcbe53e](https://github.com/xcompact3d/x3d2/tree/fcbe53e75412bcb84d7a2eb1a9554d03f9cf7842): `fix: update 2DECOMP-FFT for non-x86 NVHPC builds (#340)` dated 13th August 2026 was used instead as advised by the submitter. The pre-assessment of the GPU configuration was performed by Emily Wilkinson and Ananya Gangopadhyay of Durham University between the 8th of July and 9th October 2026.

## Disclaimers

1. This report is not a commentary on code quality, but an indicator of the quality of the current SHAREing testing methodology as of 4th September 2026.
2. The pre-assessment is only a preliminary assessment of submission suitability and does not guarantee a full assessment. It will be provided to the submitter indicating if the full assessment will be undertaken or detail reasons for rejection.

## Table of contents

- [SHAREing: Pre-assessment report of x3d2](#-shareing-pre-assessment-report-of-x3d2)
  - [Disclaimers](#disclaimers)
  - [Table of contents](#table-of-contents)
  - [1. Benchmark setup](#1-benchmark-setup)
    - [1.1 Fetch and build program](#11-fetch-and-build-program)
    - [1.2 Run benchmark](#12-run-benchmark)
    - [1.3 Reference architecture](#13-reference-architecture)
  - [2. Description of working environment](#2-description-of-working-environment)
    - [2.1 Hardware information](#21-hardware-information)
    - [2.2 Libraries and modules](#22-libraries-and-modules)
    - [2.3 Assessment tools](#23-assessment-tools)
  - [3. Compiler setup and optimisations](#3-compiler-setup-and-optimisations)
  - [4. Computational complexity and scaling](#4-computational-complexity-and-scaling)
  - [5. Memory, storage and I/O](#5-memory-storage-and-io)
  - [6. Additional comments from submitter](#6-additional-comments-from-submitter)
  - [7. Pre-assessment outcome](#7-pre-assessment-outcome)

## 1. Benchmark setup

### 1.1 Fetch and build program

Code for the CPU configuration is fetched with:

```bash
git clone https://github.com/xcompact3d/x3d2/ && cd x3d2
git checkout fcbe53e75412bcb84d7a2eb1a9554d03f9cf7842
```

The CPU version of the program can then be built as follows, yielding an executable at `build-omp/bin/xcompact`:

```bash
cmake -S . -B build-omp -DCMAKE_BUILD_TYPE=Release -DWITH_ADIOS2=ON
cmake --build build-omp -j
```

For the GPU configuration, an updated version of the code was built from the following commit:

```bash
git checkout 84c750f38af041661852dc35858c879293146db6
```

The GPU configuration, yielding an executable at `build-cuda/bin/xcompact`:

```bash

cmake -S . -B build-cuda -DCMAKE_BUILD_TYPE=Release -DCMAKE_Fortran_COMPILER=mpif90
cmake --build build-cuda -j
```

### 1.2 Run benchmark

This assessment uses the Taylor-Green benchmark which is included in the repository at `examples/TGV/input.x3d`. To run the benchmark with the MPI + OpenMP configuration:

```bash
OMP_NUM_THREADS=3 mpirun -np 8 ./build-omp/bin/xcompact examples/TGV/input.x3d
```

The following output is expected every 100 iterations:

```txt
 Time for this time step (s):  0.645300210
 time =  0.10000000000000001      iteration =         100
 enstrophy:  0.37524995458003335
 div u max mean:   8.8705084944074031E-014   4.9909836307199695E-015
```

 The MPI + GPU configuration can be run on a single GPU with:

```bash
OMP_NUM_THREADS=1 mpirun -np 1 ./build-cuda/bin/xcompact examples/TGV/input.x3d
```

Additionally, the files `monitoring.csv` and `decomp_2d_setup.log` will be generated in the IO-enabled version.

### 1.3 Reference architecture

Reference benchmark architecture for the submitted benchmark case:

- CPU: AMD EPYC 7443 24-Core Processor, 1 socket, 24 physical cores, 48 hardware threads
- NUMA: 4 NUMA domains
- GPU: NVIDIA A100.

## 2. Description of working environment

### 2.1 Hardware information

Two different clusters will be used for this assessment.
The first is the Hamilton system, and the second is Cosma, both based at Durham University.

Hamilton will be used to run the CPU-only version of the submitted code.

The hardware details for Hamilton are available [here](https://www.durham.ac.uk/research/institutes-and-centres/advanced-research-computing/hamilton-supercomputer/systems/). This information is corroborated by running `cat /proc/cpuinfo` on one of the compute nodes via an interactive session. There are 120 standard compute nodes on the system:

| Specification             | Per node                                                                     |
| ------------------------- | ---------------------------------------------------------------------------- |
| Processors                | 2 $\times$ [AMD EPYC 7702](https://www.amd.com/en/support/downloads/drivers.html/processors/epyc/epyc-7002-series/amd-epyc-7702.html)       |
| Clock speed per CPU       | 2000 MHz base clock, 3350 MHz boost clock             |
| Sockets                   | 2                                                              |
| Cores / socket            | 64                                                                |
| NUMA domains / socket     | 4                                                    |
| L1/L2 Cache size / core   | 32KB L1, 512KB L2 |
| L3 Cache size / 4-core chiplet | 16MB |
| Chiplets / NUMA domain    | 4 |
| Simultaneous Multithreading | Disabled
| RAM                       | 256GB DDR4                                           |
| Local storage             | 400GB SSD                                      |
| Interconnect              | Infiniband HDR 200GB/s high performance interconnect, with a 2.6:1 fat tree topology formed of non-blocking islands of up to 26 nodes |

Additionally, there is a high-memory partition with 2 nodes and a single GPU node in its own partition, which will not be used in this assessment.

For the heterogenous MPI + GPU configuration, the [Grace Hopper GH200 testbed on COSMA](https://cosma.readthedocs.io/en/latest/nvidiagpu.html#gh200-shareing) is used. The testbed contains two nodes, with `gn002` accessible via `ssh` and `gn003` via SLURM queue, with the following specifications:

| Specification | Per node |
| ------------------------- | ---------------------------------------------------------------------------- |
| Processor | 1 $\times$ [NVIDIA GH200 Grace Hopper Superchip](https://dam-cdn.nvd.orangelogic.com/AssetLink/tkno03fa645gb7f33w60rf200d330420.pdf) with a [72-core ARM Grace CPU](https://resources.nvidia.com/en-us-grace-cpu/data-center-datasheet?xs=1123791#page=1) and an [H100 GPU](https://www.nvidia.com/en-gb/data-center/h100/) |
| Clock speed per CPU | 3.1 GHz (base frequency) or 3.0 GHz (all-core SIMD frequency) |
| L1 Cache size | 64KB i-cache + 64KB d-cache |
| L2 Cache size | 1MB per core |
| L3 Cache size | 114MB |
| RAM | 480GB (unified) LPDDR5X |
| Interconnect | [NVLink-C2C](https://www.nvidia.com/en-gb/data-center/nvlink-c2c/) |

The MPI + GPU build is compiled on `gn002` and deployed on `gn003` (referred to as the `gracehopper` partition) for benchmarking and profiling.

### 2.2 Libraries and modules

The submitters did not indicate a preference for the compiler suite for the MPI + OpenMP build. To build this version on the assessment system, Hamilton, we need to load the following modules:

```bash
module load gcc openmpi
```

This loads GCC 11.2.0 and OpenMPI 4.1.1, along with cmake 3.26.5 being present by default. A minimum version of CMake 3.21 is required for 2DECOMP&FFT library. The modules must also be loaded for running the benchmark via interactive jobs or SLURM submission scripts.

For the GPU + MPI build on the GH200 Cosma testbeds, the NVHPC compiler suite with its bundled version of OpenMPI is used, with the submitters indicating that NVHPC's `mpif90` should be used. The latest version of the compiler suite on `gn002` is loaded with:

```bash
module load nvhpc/26.9
```

For deploying the jobs on `gn003`/`gracehopper`, the module must be loaded with:

```bash
export MODULEPATH=/opt/nvidia/hpc_sdk/modulefiles:$MODULEPATH
module load nvhpc
```

### 2.3 Assessment tools

The following tools are available on Hamilton and intended for use for the pre-assessment and high-level:

1. `darshan` 3.5.0 for I/O
2. GNU `time` (`/usr/bin/time`) for runtime and memory usage
3. NVIDIA Nsight Compute `ncu` for collecting GPU performance metrics

## 3. Compiler setup and optimisations

The submitted code uses the `cmake` build system. The invocation for the [MPI + OpenMP build](#11-fetch-and-build-program) configures the build for release, which adds the following compiler optimisation flags:

```bash
-O3 -ffast-math -funroll-loops -floop-optimize -march=native
```

The MPI + OpenMP and the MPI + GPU configurations were built on Hamilton and COSMA's `gn002` testbed successfully with the [modules specified](#22-libraries-and-modules).

## 4. Computational complexity and scaling

Submitters provided the following details (copied directly from the form):

> The main problem-size parameter is `dims_global = Nx, Ny, Nz`. For cubic cases, increasing N from 256 to 512 increases the number of grid cells by 8x.
>
> The core flow-solver work scales approximately with the number of grid cells times the number of timesteps:
>
```txt
Work ~ O(Nx * Ny * Nz * n_iters)
```

> For FFT-based Poisson solves and global transposes, the cost also includes communication and FFT complexity. For cubic grids this introduces an approximate FFT contribution of:

```txt
O(N^3 log N)
```

> plus MPI communication/transposition costs that depend on the domain decomposition and interconnect.
>
> To produce strong scaling:
>
> - keep `dims_global` fixed
> - increase MPI ranks / OpenMP threads / GPUs
> - adjust `nproc_dir` so its product matches the MPI rank count
>
> To produce weak scaling:
>
> - increase dims_global as the number of ranks/GPUs increases so that the local grid size per rank/GPU remains approximately constant
> - for example, if doubling resources in one decomposition direction, double the corresponding global grid dimension and update nproc_dir accordingly
>
> For single-node CPU benchmarks on a 24-core AMD EPYC 7443, useful configurations include:
>
> - 24 MPI ranks x 1 thread
> - 12 MPI ranks x 2 threads
> - 8 MPI ranks x 3 threads
> - 4 MPI ranks x 6 threads
>
> For GPU benchmarks, start with:
>
> - 1 MPI rank per GPU

## 5. Memory, storage and I/O

A test run of the program indicates that 4.58 GB of RAM was used for a default (256x256x256) sized run with 8 ranks decomposed 1, 2, 4 over 3000 iterations.

The benchmark writes to the console output every 100 iterations [as indicated by the submitter](#12-run-benchmark). The benchmark also outputs files totalling 0.02 MB for a run with 3000 iterations. These can be disabled by setting

```txt
n_output = 0
checkpoint_freq = 0
snapshot_freq = 0
keep_checkpoint = F
restart_from_checkpoint = F
```

or enabled for an I/O assessment run with

```txt
n_output = 10
checkpoint_freq = 100
snapshot_freq = 100
keep_checkpoint = T
```

## 6. Additional comments from submitter

Submitters provided the following details (copied directly from the form):

> The submitted Taylor-Green vortex benchmark is intended primarily as a performance benchmark for x3d2 rather than a full physics-validation submission. It exercises the main computational patterns of the code: compact finite-difference operators, pressure projection/Poisson solve, MPI communication/domain decomposition, OpenMP CPU execution, CUDA GPU execution, and optional ADIOS2 I/O.
>
> The primary quantity of interest is the timestep loop performance, excluding or at least separating initial setup where possible.
> x3d2 prints timing information including averaged time per timestep. This is the preferred metric for solver performance because it focuses on the repeated compute phase rather than one-off setup costs.
> For most performance comparisons, please report:
>
> - total wall time for the full run
> - averaged time per step
> - number of iterations
> - backend and parallel configuration

## 7. Pre-assessment outcome

The submitted program is suitable to proceed to a high-level assessment.
