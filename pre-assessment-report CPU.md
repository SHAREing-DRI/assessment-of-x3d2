---
License: Copyright © 2026 Durham University, SHAREing Project, MIT Licensed
Creator: Andrew Naden
Contributors: Andrew Naden; Ananya Gangopadhyay
Summary: Pre-assessment report template document
---

# Pre-assessment of x3d2

## Assessment objective

An external submission of x3d2 ([github](https://github.com/xcompact3d/x3d2/), commit 84c750f38af041661852dc35858c879293146db6 `Faster OpenMP Reorder and Accumulation Kernels (#329)`) was made on the 8th of July 2026 to the SHAREing team at Durham.
This assessment is performed by Emily Wilkinson of Durham University on the 8th of July 2026.


## Disclaimers

1. This report is not a commentary on code quality.
2. The pre-assessment is only a preliminary assessment of submission suitability and does not guarantee a full assessment. It will be provided to the submitter indicating if the full assessment will be undertaken or detail reasons for rejection.

## Table of contents

- [ ] [1: Benchmark setup](#1-benchmark-setup)
- [ ] [2: Description of working environment](#2-description-of-working-environment)
- [ ] [3: Compiler setup and optimisations](#3-compiler-setup-and-optimisations)
- [ ] [4: Computational complexity and scaling](#4-computational-complexity-and-scaling)
- [ ] [5: Memory, storage and I/O](#5-memory-storage-and-io)
- [ ] [6: Additional comments from submitter](#6-additional-comments-from-submitter)
- [ ] [7: Pre-assessment outcome](#7-pre-assessment-outcome)

## 1: Benchmark setup

### Fetch and build program

The submitter has requested an assessment of x3d2, commit 84c750f38af041661852dc35858c879293146db6. It can be fetched from github as follows:

```bash
git clone https://github.com/xcompact3d/x3d2/ && cd x3d2
git checkout 84c750f38af041661852dc35858c879293146db6
```

The CPU version of the program can then be built as follows, yielding an executable at `build-omp/bin/xcompact`:

```bash

cmake -S . -B build-omp -DCMAKE_BUILD_TYPE=Release -DWITH_ADIOS2=ON
cmake --build build-omp -j
```

### Fetch and run benchmark

The benchmark is included in the same git repository as the program. The benchmark file selected is the Taylor-Green 

To run the benchmark:

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

Additionally, the files `decomp_2d_setup.log` and `monitoring.csv` will be generated in the IO-enabled version.

### Reference architecture

Reference benchmark architecture for the submitted benchmark case:
- CPU: AMD EPYC 7443 24-Core Processor, 1 socket, 24 physical cores, 48 hardware threads
- NUMA: 4 NUMA domains
- GPU: NVIDIA A100.

## 2: Description of working environment

### Hardware information

Two different clusters will be used for this assessment.
The first is the Hamilton system, and the second is Bede, both based at Durham University.

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


### Libraries and modules

In order to build the CPU version on the assessment system, Hamilton8, we need to load the following modules:

```bash
module load gcc openmpi
```

This loads GCC 11.2.0 and OpenMPI 4.1.1, along with cmake 3.26.5 being present by default.
A minimum version of cmake 3.21 is required by 2decomp.

### Assessment tools

The following tools are available on Hamilton and intended for use for the high-level assessment:
1. likwid 5.5.1
2. darshan 3.5.0
3. GNU time (/usr/bin/time)

## 3: Compiler setup and optimisations

The submitted code uses the `cmake` build system.
The configuration invocation
```
cmake -S . -B build-omp -DCMAKE_BUILD_TYPE=Release -DWITH_ADIOS2=ON
```
configures the build for release, which adds the following compiler optimisation flags for the CPU build:
```
-O3 -ffast-math -funroll-loops -floop-optimize -march=native
```

No compiler or library versions were specified and therefore a set which successfully compiled the program was selected and is described in the [description of the working environments](#2-description-of-working-environment).

## 4: Computational complexity and scaling

### Submitted comments (copied direct from form)

The main problem-size parameter is dims_global = Nx, Ny, Nz. For cubic cases, increasing N from 256 to 512 increases the number of grid cells by 8x.

The core flow-solver work scales approximately with the number of grid cells times the number of timesteps:
```
Work ~ O(Nx * Ny * Nz * n_iters)
```
For FFT-based Poisson solves and global transposes, the cost also includes communication and FFT complexity. For cubic grids this introduces an approximate FFT contribution of:
```
O(N^3 log N)
```
plus MPI communication/transposition costs that depend on the domain decomposition and interconnect.

To produce strong scaling:
- keep dims_global fixed
- increase MPI ranks / OpenMP threads / GPUs
- adjust nproc_dir so its product matches the MPI rank count

To produce weak scaling:
- increase dims_global as the number of ranks/GPUs increases so that the local grid size per rank/GPU remains approximately constant
- for example, if doubling resources in one decomposition direction, double the corresponding global grid dimension and update nproc_dir accordingly

For single-node CPU benchmarks on a 24-core AMD EPYC 7443, useful configurations include:
- 24 MPI ranks x 1 thread
- 12 MPI ranks x 2 threads
- 8 MPI ranks x 3 threads
- 4 MPI ranks x 6 threads

For GPU benchmarks, start with:
- 1 MPI rank per GPU

## 5: Memory, storage and I/O

>[!IMPORTANT]
> Comment on the expected in memory size of the program at runtime, including data. An estimate of this information should be provided as part of the submission. For jobs submitted to Hamilton as part of early assessment, the Hamilton dashboard can be used to gauge memory usage (see [Hamilton Portal Performance](https://www.durham.ac.uk/research/institutes-and-centres/advanced-research-computing/hamilton-supercomputer/usage/portal/performance/)).

4.58 GB

>[!IMPORTANT]
> Comment on the expected storage requirements of the program, are there large amounts of temporary files (either in quantity or in total size)? An estimate of this information should be provided as part of the submission. A program that produces a large amount of temporary checkpoint files should have checkpoints turned off where possible.

The benchmark also outputs files totalling `0.02`MB.

>[!IMPORTANT]
> Comment on the expected output, including when the I/O is performed, and your observations when running the benchmark. This output should be minimal when testing the working performance of the program rather than the I/O saturation. Excessive I/O will result in an inaccurate performance assessment and may result in rejection.

The benchmark writes to the console output every `<n>` iterations [as indicated by the submitter](#fetch-and-run-benchmark).

## 6: Additional comments from submitter

>[!IMPORTANT]
> Include any additional information from the submitter that does not fit the previous sections.

## 7: Pre-assessment outcome

>[!IMPORTANT]
> Indicate whether the assessment will proceed to the high-level stage. If the assessment is rejected here, comment on why and how to proceed.
