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

and the GPU version can be built as follows, yielding an executable at `build-cuda/bin/xcompact`:

```bash
cmake -S . -B build-cuda -DCMAKE_BUILD_TYPE=Release -DWITH_ADIOS2=ON
cmake --build build-cuda -j
```

Additionally, the `-march=native` compile flag is not recognised by the Bede version of `nvfortran` so this flag must be removed manually from the `2decomp-fft` cmake lists in order to build successfully.

# *** GPU BUILD DOESN't WORK YET! ***

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

>[!IMPORTANT]
> Add details of the reference architecture as provided by the submitter. Add any relevant details you may find regarding the architecture/system online. If the same system is accessible to you for the assessment, then indicate that here and detail the information in the [next section](#hardware-information).

Reference benchmark architecture for the submitted benchmark case:
- CPU: AMD EPYC 7443 24-Core Processor, 1 socket, 24 physical cores, 48 hardware threads
- NUMA: 4 NUMA domains
- GPU: NVIDIA A100.

## 2: Description of working environment

### Hardware information

Two different clusters will be used for this assessment.
The first is the Hamilton system, and the second is Bede, both based at Durham University.

Hamilton will be used to run the CPU-only version of the submitted code, while Bede will be used to run the heterogeneous CPU+GPU version of the code.

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

The hardware details for Bede are available [here](https://bede-documentation.readthedocs.io/en/latest/hardware/index.html). This information is corroborated by running `cat /proc/cpuinfo` on one of the compute nodes via an interactive session. There are 32 standard GPU compute nodes on the system:

| Specification             | Per node                                                                     |
| ------------------------- | ---------------------------------------------------------------------------- |
| Processors                | 2 $\times$ [POWER9 CPU]()       |
| Clock speed per CPU       | 2700 MHz base clock, 3800 MHz boost clock              |
| Sockets                   | 2                                                              |
| Cores / socket            | 40                                                                |
| NUMA domains / socket     | 1                                                    |
| L1/L2 Cache size / core   | 32KB L1i/32KB L1d, 512KB L2 |
| L3 Cache size / 4-core chiplet | 10MB |
| Simultaneous Multithreading | 4 threads / core |
| GPU                       | 4 $\times$ [Nvidia Tesla V100 32GB](https://images.nvidia.com/content/technologies/volta/pdf/volta-v100-datasheet-update-us-1165301-r5.pdf) |
| GPU interconnect          | NVLink 2.0 |
| RAM                       | 512GB DDR4                                           |
| Interconnect              | 2 $\times$ Mellanox EDR (100Gbit/s) InfiniBand ports, organised in a 2:1 block fat tree topology |

Additionally, there is an infer partition with Tesla T4 GPUs and 7 gracehopper nodes in a separate partition, which will not be used in this assessment.


### Libraries and modules

In order to build the CPU version on the assessment system, Hamilton8, we need to load the following modules:

```bash
module load gcc openmpi cmake/3.24.3
```

This loads GCC 11.2.0 and OpenMPI 4.1.1.

In order to build the GPU version on the assessment system, Bede, we need to load the following modules:

```bash
module load nvhpc/23.1 openmpi
```

### Assessment tools

> [!IMPORTANT]
>
> 1. Limit pre-assessment tools to those with very low runtime. Mostly just focus on whether the program is running as expected. Do not assess the results of the benchmark for correctness as that requires domain-specific knowledge.
> 2. High level assessment tools and techniques which are expected to be useful, like global measures such as wall time.
> 3. If additional information is provided, you can address the low-level assessment that may be required, and if you may require privileges on the system. Only address this section if you confident that enough domain information has been provided, with respect to scaling of the compute and memory with the problem size.

2. High-level assessment:
   - abc

## 3: Compiler setup and optimisations

>[!IMPORTANT]
> Based on the compilation information provided by submitter, comment on the following (where applicable):
>
> - [x] package manager (e.g. `spack`)
> - [x] build toolchain (e.g. `cmake`)
> - [x] main compiler version (e.g. GCC 11)
> - [x] compiler optimisations (e.g. -O3, `--fast-math`)
> - [ ] additional accelerator libraries and versions (e.g. SYCL revision 11, Kokkos 5.1)
> - [ ] any feature sets which are toggled on (e.g. vectorisation)
>
> Add additional information about the impact of optimisations on convergence or correctness of results if provided by the submitter.  If there are any issues with compatibility on the machine you are testing on, or any build issues experienced, provide details


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

>[!IMPORTANT]
> Comment on the possibility of scaling the problem up and down, both in strong (changing number of work units e.g. CPUs, but keeping the problem size constant) and weak (changing the problem size but keeping number of work units the same) contexts. Add any information provided by the submitter regarding the scaling of _computation (i.e. work)_, _memory_ and _execution time_ as the problem size or work units are increased.
>
> If there is existing scaling information (graphs or raw data) available, attach it to this report or add links to access it.

The `<parameter>` value can be varied to increase the problem size for scaling tests.

## 5: Memory, storage and I/O

>[!IMPORTANT]
> Comment on the expected in memory size of the program at runtime, including data. An estimate of this information should be provided as part of the submission. For jobs submitted to Hamilton as part of early assessment, the Hamilton dashboard can be used to gauge memory usage (see [Hamilton Portal Performance](https://www.durham.ac.uk/research/institutes-and-centres/advanced-research-computing/hamilton-supercomputer/usage/portal/performance/)).

According to the submitter, the expected memory required for the benchmark is `<memory_size>`GB.

>[!IMPORTANT]
> Comment on the expected storage requirements of the program, are there large amounts of temporary files (either in quantity or in total size)? An estimate of this information should be provided as part of the submission. A program that produces a large amount of temporary checkpoint files should have checkpoints turned off where possible.

The benchmark also outputs files totalling `<storage_size>`MB.

>[!IMPORTANT]
> Comment on the expected output, including when the I/O is performed, and your observations when running the benchmark. This output should be minimal when testing the working performance of the program rather than the I/O saturation. Excessive I/O will result in an inaccurate performance assessment and may result in rejection.

The benchmark writes to the console output every `<n>` iterations [as indicated by the submitter](#fetch-and-run-benchmark).

## 6: Additional comments from submitter

>[!IMPORTANT]
> Include any additional information from the submitter that does not fit the previous sections.

## 7: Pre-assessment outcome

>[!IMPORTANT]
> Indicate whether the assessment will proceed to the high-level stage. If the assessment is rejected here, comment on why and how to proceed.
