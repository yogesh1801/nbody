# GPU Implementation of Direct \$N\$-Body Simulation

This project provides multiple implementations of the direct \$N\$-body simulation method, optimized for GPUs using different programming approaches.

## Overview

This repository compares and evaluates the performance of direct \$N\$-body simulations across several development environments:

* **CPU (C++)**: A naive baseline implementation
* **CUDA C++**: [GPU implementation using CUDA](cuda.md)
* **OpenACC**: [GPU offloading with OpenACC directives](openacc.md)
* **OpenMP**: [GPU offloading with OpenMP target directives](openmp.md)
* **C++17 (stdpar)**: [GPU offloading with standard parallelism features](stdpar.md)
* **Solomon**: [GPU offloading using Solomon, a directive-based GPU integration library](solomon.md)

Licensed under the **MIT License** (see `LICENSE.txt`).
© 2022 Yohei MIKI

---

## Getting the Source Code

Clone the repository with submodules:

```sh
git clone --recurse-submodules git@github.com:ymiki-repo/nbody.git
```

---

## Building the Project

### Prerequisites

* **CMake >= 3.20** (NVIDIA HPC SDK support begins with 3.20)
* **Boost**
* **HDF5**

### Recommended Modules

#### Miyabi-G (NVIDIA HPC SDK)

```sh
module purge
module load nvidia   # NVIDIA HPC SDK
module load hdf5
```

#### Miyabi-G (CUDA)

```sh
module purge
module load cuda
module load use /work/share/opt/modules/lib   # required for hdf5
module load hdf5
```

#### Wisteria/BDEC-01 (Aquarius, NVIDIA HPC SDK)

```sh
module purge
module load cmake
module load nvidia/22.7
module load hdf5
```

#### Wisteria/BDEC-01 (Aquarius, CUDA)

```sh
module purge
module load cmake
module load cuda
module load gcc      # required for hdf5
module load hdf5
```

### Optional (Visualization)

* [Julia](https://julialang.org/)
* [VisIt](https://wci.llnl.gov/simulation/computer-codes/visit)

### Configuration

Using **ccmake (GUI)**:

```sh
cmake -S . -B build
cd build
ccmake -S ..
```

Using **command line only**:

```sh
cmake -S . -B build [options]
cd build
```

Reconfigure from scratch (**CMake 3.24+**):

```sh
cmake --fresh -S . -B build [options]
```

### Key CMake Options

| Option                                               | Description                                            |
| ---------------------------------------------------- | ------------------------------------------------------ |
| `-DBENCHMARK_MODE=ON`                                | Enable benchmark mode                                  |
| `-DCALCULATE_POTENTIAL=ON`                           | Compute gravitational potential (default: ON)          |
| `-DFP_L=[32/64/128]`                                 | Floating-point precision (low, default: 32)            |
| `-DFP_M=[32/64/128]`                                 | Floating-point precision (medium, default: 32)         |
| `-DFP_H=[64/128]`                                    | Floating-point precision (high, default: 64)           |
| `-DHERMITE_SCHEME=ON`                                | Use 4th-order Hermite scheme (default: OFF = Leapfrog) |
| `-DSIMD_BITS=[256/512/1024]`                         | SIMD width (default: 512)                              |
| `-DUSE_CUDA=ON`                                      | Enable CUDA GPU backend                                |
| `-DGPU_EXECUTION=ON`                                 | Enable GPU execution (default: ON)                     |
| `-DNTHREADS=[32..1024]`                              | Threads per thread block (default: 256)                |
| `-DUNROLL=[1..1024]`                                 | Loop unroll count (default: 128)                       |
| `-DRELAX_RSQRT_ACCURACY=ON`                          | Relax reciprocal sqrt precision (NVIDIA HPC SDK only)  |
| `-DEXERCISE_MODE=ON`                                 | Enable exercise/demo mode                              |
| `-DTARGET_GPU=[NVIDIA_CC90/NVIDIA_CC80/NVIDIA_CC70]` | Target GPU architecture (default: CC90, Hopper)        |
| `-DTIGHTLY_COUPLED_CPU_GPU=ON`                       | Enable NVLink-C2C CPU–GPU fusion (default: ON)         |

### Build

```sh
ninja   # preferred (if ninja-build is installed)
make    # fallback (if ninja is not available)
```

---

## Running Simulations

Check available options:

```sh
./nbody --help
```

Examples:

* Run with **default settings**:

  ```sh
  ./nbody
  ```

* Set **number of particles**:

  ```sh
  ./nbody --n 65536
  ```

* Set **time steps and step size**:

  ```sh
  ./nbody --n 65536 --step 100 --dt 0.01
  ```

* Specify **output file**:

  ```sh
  ./nbody --n 65536 --output result.h5
  ```

---

## Visualization

Results can be visualized with Julia or VisIt.

Example (Julia):

```sh
julia visualize.jl result.h5
```

---

## Benchmarking & Performance

To run benchmarks:

```sh
./nbody --benchmark
```
Performance metrics include:

* Runtime per step
* Achieved GFLOPS
* Scaling efficiency across GPU architectures

